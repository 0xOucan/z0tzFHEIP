---
fheip: 0007
title: Ciphertext integrity binding in signed digests
description: Define a canonical `"CoFHE.Bind/1"` domain-separated digest transform and an EIP-712 typed-struct helper so contracts that accept relayer-submitted `InEuintN` inputs can bind the ciphertext handle into their authorization digest, defeating amount-substitution attacks without each dapp inventing its own scheme.
author: 0xOucan (@0xOucan)
discussions-to: TBD
status: Draft
type: Standards Track
category: Core
created: 2026-04-17
requires: FHE.sol (cofhe-contracts), EIP-712, EIP-191
---

## Abstract

A relayer-submitted FHE operation — "spend 10 tokens," "transfer this encrypted amount" — is a two-part object: (a) a user-signed authorization over the intent, and (b) an `InEuintN` ciphertext encoding the amount. If the authorization digest doesn't include the ciphertext handle, a malicious relayer can substitute a different `InEuintN` (e.g., "spend 10" becomes "spend 10 million") because the signature only binds the intent, not the amount. Every FHE contract accepting relayer-submitted encrypted amounts must address this. This FHEIP canonicalizes the pattern: a one-line `FHE.bindCtHash(authDigest, ctHash)` helper, a matching EIP-712 typed struct, and a documentation section in FHE.sol that names the substitution attack.

## Motivation

The attack is subtle but trivial to execute. Consider:

```solidity
function spend(SpendOp calldata op) external {
    bytes32 digest = keccak256(abi.encode(op.oldId, op.amount.ctHash, op.deadline));
    //                                      ^^^^^^^^^^^^^^^^^^^^
    //                                      bound                  <-- GOOD
    require(P256.verify(digest, op.sig, op.pkX, op.pkY));
    euint64 amount = FHE.asEuint64(op.amount);
    // ...
}
```

vs:

```solidity
function spend(SpendOp calldata op) external {
    bytes32 digest = keccak256(abi.encode(op.oldId, op.deadline));
    //                                    ^^^^^^^^^^^^^^^^^^^^^^
    //                                    NOT bound             <-- BAD
    require(P256.verify(digest, op.sig, op.pkX, op.pkY));
    euint64 amount = FHE.asEuint64(op.amount);  // relayer chose this freely
    // ...
}
```

The user signed "spend something by deadline" — they never committed to *what* ciphertext. A relayer holding both a 10-token and a 10M-token encryption of valid-for-this-user shape can choose which to forward.

Z0tz identified this as a security finding during its V6.5 audit and fixed it by adding `op.amount.ctHash` to the keccak preimage of the spend digest. The fix is mechanically simple — one extra field in one `abi.encode` call. The problem is that the attack's existence is not documented anywhere in FHE.sol's user-facing API: a dapp author reading the cofhe-contracts tutorials has no signal that the ciphertext must appear in the signed digest. Canonicalizing the bind as a library helper plus a documentation commitment converts a reviewer-caught footgun into an unmissable compile-time signal.

### Use cases

- Every relayer-mediated FHE operation (4337 UserOps, meta-transactions, gasless sends)
- Smart-account flows where the account owner signs an authorization and a relayer bundles it with a user-encrypted amount
- Multi-step flows (ledger.spend with a nested `InEuintN` amount, FHERC-20 `confidentialTransferFrom`-style ops)

### Non-goals

- Preventing the relayer from choosing *not* to submit (censorship is a different problem).
- Solving ciphertext authorship (anyone with the user's public key can construct an `InEuintN`; the ctHash binding only ensures the *specific* ciphertext the user intended).

## Specification

### 1. Domain-separated bind helper

```solidity
library FHE {
    /// Canonical ciphertext binding.
    /// `boundDigest = keccak256(abi.encode("CoFHE.Bind/1", authDigest, ctHash))`
    function bindCtHash(bytes32 authDigest, bytes32 ctHash) internal pure returns (bytes32);
}
```

Contracts using this helper signal that their authorization path is aware of substitution attacks. Static analyzers can check for `FHE.bindCtHash` presence in any function that accepts an `InEuintN`.

### 2. EIP-712 typed-struct helper

For contracts using EIP-712 signatures, add the following type to the signed struct:

```
CtHashBinding(bytes32 authDigest,bytes32 ctHash)
```

And in the composite type:

```
SpendOp(bytes32 oldId,...,CtHashBinding amount,...)
CtHashBinding(bytes32 authDigest,bytes32 ctHash)
```

Where `amount` has `authDigest` set to the hash of the non-ciphertext fields (oldId, deadline, etc.) and `ctHash` set to the ciphertext handle. The EIP-712 encoder traverses the nested struct automatically; the binding emerges from the type hash.

### 3. Multi-handle variant

For authorizations covering multiple ciphertext handles (batch spends, multi-output flows):

```solidity
library FHE {
    function bindCtHashes(bytes32 authDigest, bytes32[] calldata ctHashes) internal pure returns (bytes32);
    // = keccak256(abi.encode("CoFHE.Bind/1", authDigest, keccak256(abi.encodePacked(ctHashes))))
}
```

The inner `keccak256(abi.encodePacked(ctHashes))` keeps the binding order-sensitive: reordering ciphertexts changes the bound digest, preventing a relayer from swapping "slot 0 and slot 1" amounts if that matters to the dapp.

### 4. FHE.sol documentation commitment

FHE.sol's inline NatSpec on `asEuint64` / `asEuint128` / etc. MUST include a "Security" paragraph naming the substitution attack and pointing at `bindCtHash`. Example:

```solidity
/// @notice Converts an externally-provided `InEuint64` into an internal `euint64` handle.
/// @dev SECURITY: if this function is called from a path where the caller's
///      authorization digest does NOT include the `InEuint64.ctHash`, a relayer
///      can substitute a different ciphertext of the same shape. See FHEIP-0007
///      and `FHE.bindCtHash` for the canonical mitigation.
function asEuint64(InEuint64 calldata input) internal returns (euint64);
```

## Rationale

**Why a domain tag `"CoFHE.Bind/1"`?** Without it, an attacker could construct a bound digest that collides with another application's signed digest (signature reuse across protocols). The tag + version string follows the convention used by FHEIP-0001 for viewer-address derivation.

**Why `keccak256` and not an FHE primitive?** `bindCtHash` runs on plaintext bytes (the ctHash IS plaintext — only the *value* behind it is encrypted). Using keccak matches the rest of Ethereum's signing infrastructure and lets existing wallets sign with no changes.

**Why put the bind AFTER computing `authDigest`?** Composability: apps that already have a signed EIP-712 struct for the non-ciphertext fields can call `bindCtHash(existingDigest, ctHash)` as a one-liner without restructuring their type hash. Apps building from scratch can inline the bind as a nested struct per §2 for cleaner typed-struct presentation.

**Why offer both a helper and a typed-struct pattern?** The helper is for terse implementations (three lines inside a function body). The typed struct is for wallets that render EIP-712 messages to users — the nested `CtHashBinding` field shows up in the signing UI, which is the right surface for a user to notice they're authorizing a specific ciphertext.

## Backwards Compatibility

Additive. Existing contracts that do not bind continue to work (they remain vulnerable to substitution, but the proposal doesn't break them). New contracts opt in by calling `bindCtHash` or adding the nested struct. Wallet signing UIs render the nested struct natively with no changes.

## Security Considerations

1. **Domain tag uniqueness.** `"CoFHE.Bind/1"` MUST be treated as a reserved string. Future binding schemes (post-quantum hashes, different handle formats) use `"CoFHE.Bind/2"`, etc. Never reuse.
2. **ctHash is public.** Binding in the digest prevents substitution but does NOT hide the ciphertext handle — ctHashes are already visible on-chain via event logs. Security comes from the user committing to *this* handle, not from secrecy.
3. **`ctHashes` ordering in batches.** §3's concatenation is order-sensitive. Dapps where amount ordering does not matter (set-like batches) SHOULD sort `ctHashes` canonically before hashing to avoid signing confusion.
4. **Relayer censorship is orthogonal.** A relayer that refuses to submit is outside this FHEIP's scope. Dapps that want censorship resistance must structure their flow around alternative submission paths (multiple relayers, direct user submission as fallback).
5. **Incremental deployment risk.** During the rollout window, a dapp's old functions (pre-bind) and new functions (post-bind) coexist. Dapps MUST NOT keep pre-bind functions reachable for signers who have been educated about the bind — the old path is still substitutable.
6. **Wallet UX.** Wallets rendering EIP-712 must NOT hide the nested `CtHashBinding`. A user who signs without seeing the ctHash has no meaningful defense against substitution. This is a wallet policy, not a protocol mechanism, but the typed-struct shape enables it.

## Reference Implementation

- Z0tz's current binding in the ledger spend digest: [`Z0tzPrivateLedger.sol` lines 210-228](https://github.com/0xOucan/Z0tz/blob/main/contracts/contracts/ledger/Z0tzPrivateLedger.sol#L210-L228) — the `op.amount.ctHash` field at line 224 is included in the keccak preimage alongside `op.oldId`, `op.newId`, `op.newPubkeyHash`, `op.newViewer`, `op.action`, `op.destLedgerId`, `op.destPubkeyHash`, `op.destViewer`, `op.destAddress`, `op.nonce`, and `op.deadline`. Removing line 224 is the substitution attack in one diff.
- EIP-712 nested-struct typing example: the existing `Permit` struct in `fhenix-confidential-contracts/contracts/FHERC20Permit.sol` shows how OpenZeppelin handles nested typed data, which §2's `CtHashBinding` sub-struct mirrors.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
