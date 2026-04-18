---
fheip: 0006
title: Permit V2 — rotation, batch, pluggable auth-scheme
description: Redefine the CoFHE permit as a versioned struct with a monotonic rotation counter, a batch-of-ctHashes field, and an opaque `authData` slot typed by the FHEIP-0005 auth-scheme registry, so multi-value decryption, key rotation, and new auth methods compose without per-dapp permit forks.
author: 0xOucan (@0xOucan)
discussions-to: TBD
status: Draft
type: Standards Track
category: SDK
created: 2026-04-17
requires: FHE.sol (cofhe-contracts), FHEIP-0001, FHEIP-0005
---

## Abstract

Today's CoFHE permit is a single-signer EIP-712 struct authorizing decryption of a single ciphertext handle. Apps that need to decrypt many handles (portfolio views, ledger balances across slots) sign and store one permit per handle, creating N-way storage and N-way refresh logic. Apps that need key rotation have no standard mechanism — they must issue a new permit and hope the verifier honors it over the old. Apps that want new auth methods (FHEIP-0001 P-256 passkeys, FHEIP-0005 ERC-1271 smart accounts) have no slot to drop the scheme into. This FHEIP defines **PermitV2**: a versioned struct that carries a batch of ctHashes, a monotonic rotation counter, and an opaque `authData` slot typed by the FHEIP-0005 auth-scheme registry.

## Motivation

### The single-handle permit explodes at scale

A wallet viewing N encrypted balances signs N permits (one per ctHash) or runs a custom batching helper. Every wallet implements this differently. Multi-tab sync fragments.

### There's no rotation story

`FHE.allow(handle, viewer)` grants permanent decryption rights; FHEIP-0004 adds `revokeAllow`, but the permit itself has no freshness indicator. If a user revokes an old viewer and issues a new permit, a verifier caching the old permit may accept it after the revoke tx confirms. The permit wire format has no "superseded by version N" field.

### Every new auth scheme forks the permit struct

FHEIP-0001 (P-256 passkeys) needs a permit with `pubX`/`pubY` fields. FHEIP-0005 adds ERC-1271 smart-account auth. Rolling either into the current permit shape requires a hard-fork of `PermitStruct` and breaks all existing verifiers. A pluggable shape lets both be dropped in by scheme id.

### Use cases

- Wallet portfolio views decrypting 10+ balances per refresh
- Passkey wallets rotating viewer keys per-device without rotating the whole account
- Lending / DeFi where a permit authorizes access to multiple collateral ciphertexts in one user action
- Indexers / off-chain analytics tools that need to hold a single long-lived permit across many handles

### Non-goals

- Replacing on-chain `FHE.allow` semantics.
- Defining new signature algorithms (that's FHEIP-0005's registry).
- Storing permits on-chain (permits remain off-chain; on-chain verification is covered by the verify helper in §3).

## Specification

### 1. `PermitV2` struct

```solidity
struct PermitV2 {
    uint16            version;          // MUST be 2
    uint32            rotationCounter;  // monotonic per-issuer, see §2
    address           issuer;           // 20-byte canonical identity (see §4)
    bytes32[]         ctHashes;         // batch of handles authorized
    address[]         consumers;        // contracts allowed to use decryption
    uint256           deadline;         // unix seconds
    bytes32           authSchemeId;     // FHEIP-0005 registry id (left-padded UTF-8, ≤32 bytes)
    bytes             authData;         // scheme-specific opaque payload
}
```

- `ctHashes.length` MUST be ≥ 1. Upper bound is implementation-defined; verifiers MUST accept at least 32.
- `consumers` MAY be empty, meaning "any consumer." Non-empty `consumers` restrict which contract addresses may present this permit in a decryption request.
- `authSchemeId` is a UTF-8 string left-padded to 32 bytes (e.g. `"p256-v1"` → `0x0000...700323...`). This keeps it comparable via `bytes32` storage and maps directly to FHEIP-0005's registry.
- `authData` is an arbitrary bytes blob. Shape defined by the scheme.

### 2. Rotation counter

A verifier (threshold network, on-chain contract, SDK cache) MUST track, per `issuer`, the highest `rotationCounter` it has ever accepted. A presented permit is valid only if:

- `rotationCounter ≥ highestAccepted[issuer]`, AND
- the permit otherwise verifies.

On accept, the verifier updates `highestAccepted[issuer] = rotationCounter`.

This gives rotation-like semantics without requiring on-chain state: the issuer signs a new permit with `rotationCounter + 1` and any verifier that sees the new permit stops honoring permits with lower counters. Revocation is a special case (issue `rotationCounter + 1` over empty `ctHashes`).

### 3. On-chain verification helper

```solidity
library FHE {
    /// Verifies a PermitV2 entirely from calldata. Returns true iff
    ///  - version == 2
    ///  - deadline >= block.timestamp
    ///  - authSchemeId is supported and authData verifies per the scheme
    ///  - rotationCounter is not below the stored floor for this issuer
    ///  - msg.sender is in `consumers` (or consumers.length == 0)
    /// Updates the stored rotation floor on success.
    function verifyPermitV2(PermitV2 calldata p) internal returns (bool);

    /// View variant: verifies without updating state. For read-only flows
    /// (`decryptForView`) where rotation state updates happen elsewhere.
    function verifyPermitV2View(PermitV2 calldata p) internal view returns (bool);
}
```

Supported schemes are stored in a scheme-verifier registry contract maintained by the CoFHE team. Adding a new scheme is a registry write, not a Core upgrade.

### 4. Canonical EIP-712 encoding for `ethsign`-family schemes

For the common case of secp256k1 EIP-712 signatures, the type hash is:

```
PermitV2(uint16 version,uint32 rotationCounter,address issuer,bytes32[] ctHashes,address[] consumers,uint256 deadline,bytes32 authSchemeId)
```

Note `authData` is excluded from the type hash — the signature IS the `authData` for ethsign schemes, so including it would be circular. Other schemes (P-256, ERC-1271) define their own canonical hash in FHEIP-0005.

Domain separator: standard EIP-712 with `name = "CoFHE.PermitV2"`, `version = "1"`, `chainId = <chainId>`, `verifyingContract = <ACL or scheme verifier address>`.

### 5. SDK API

```typescript
// @cofhe/sdk/permits/v2
interface PermitV2Options {
  issuer: Hex | PasskeyHandle;
  ctHashes: Hex[];
  consumers?: Address[];
  deadline: number;
  authScheme: SchemeId;   // registry id
}

createPermitV2(opts: PermitV2Options): Promise<PermitV2>;
verifyPermitV2(p: PermitV2, chainId: number): Promise<boolean>;
rotatePermitV2(old: PermitV2, delta?: Partial<PermitV2Options>): Promise<PermitV2>;
```

SDK storage is pluggable (`PermitStore` interface) so browser / Node / React-Native implementations can back permits to IndexedDB / file system / SecureStorage respectively.

## Rationale

**Why `uint16` version?** We do not expect more than ~65k permit revisions over the protocol's lifetime. A full `uint256` would waste 31 bytes in every calldata.

**Why `uint32` rotation counter?** 4 billion rotations per issuer is enough for any realistic pattern; keeps the struct compact.

**Why separate `authSchemeId` from `authData`?** Separating the discriminator from the payload lets verifiers route to the right scheme handler without parsing `authData`. Matches the HTTP `Content-Type` + body pattern.

**Why is `consumers` optional?** Some permits (public read tokens, indexer APIs) are intentionally unrestricted. Defaulting to empty-means-any avoids forcing apps to enumerate every consumer contract.

**Why is on-chain `verifyPermitV2` part of Core?** Threshold-network verification is off-chain, but composable DeFi (e.g., a lending market presenting a user's permit to borrow against encrypted collateral) needs on-chain verification. Both paths share the same struct and rotation floor, so the Core helper ensures consistency.

## Backwards Compatibility

Additive. The existing `Permit` struct continues to work; v2 is a parallel type. The SDK exports `@cofhe/sdk/permits` (v1) and `@cofhe/sdk/permits/v2` (v2). Verifiers MUST support both during a deprecation window; permits with `version == 1` use v1 semantics, `version == 2` use v2.

## Security Considerations

1. **Rotation-counter griefing.** A malicious relayer could submit a far-future `rotationCounter` to lock out the legitimate issuer. Mitigation: verifiers cap single-jump increments (e.g., `newCounter - oldCounter ≤ 2^16`) and reject larger jumps unless accompanied by a domain-separated "reset" signature. The SDK MUST NOT let applications issue huge counter jumps by accident.
2. **Replay across chains.** The EIP-712 `chainId` in the domain binds the permit to a chain. Cross-chain use requires re-signing. Do not strip chainId from the domain.
3. **Batch decryption amplifies permit compromise.** A leaked PermitV2 grants decryption rights on every ctHash it covers. Wallets SHOULD scope permits narrowly (per-app, per-view) rather than signing one omnibus permit.
4. **Consumer restriction is not sandbox.** Listing a consumer contract in `consumers` only says "this permit MAY be presented from there." The consumer contract is still responsible for the threshold-network request and could be compromised to leak decryption results.
5. **Rotation during pending requests.** If the issuer rotates between a client submitting a v2 permit and the TN verifying it, the TN will reject. Clients MUST treat `UNAUTHORIZED` after rotation as "re-sign with new counter" rather than a user-auth failure.
6. **`authData` length.** Implementations MUST cap `authData` length (e.g., 8 KB) to bound verification cost. Schemes needing larger payloads must use a hash-and-fetch pattern.

## Reference Implementation

- Current single-handle permit: `cofhesdk/packages/sdk/permits/types.ts` — the `Permit` interface declared at line 32 with `issuerSignature: Hex` at line 83 as a single-signer / single-issuer field; no batch, no rotation counter.
- Z0tz's bespoke permit-like rotation: the ledger-entry rotation in [`Z0tzPrivateLedger.sol` spend function, lines 201-298](https://github.com/0xOucan/Z0tz/blob/main/contracts/contracts/ledger/Z0tzPrivateLedger.sol#L201-L298) implements nonce-incrementing rotation at the ledger-entry level (`oldId.nonce + 1 → newId` with pubkeyHash / viewer rotation). Pulling this monotonic-nonce pattern into the permit struct generalizes it from a ledger-only feature to every authorization object.
- Batch decryption pattern prototyped in the Z0tz wallet: [`gui/src/main/ipc-handlers.ts`](https://github.com/0xOucan/Z0tz/blob/main/gui/src/main/ipc-handlers.ts) — currently decrypts balances one-permit-at-a-time with a local cache; PermitV2's `ctHashes[]` field would collapse this to one permit per refresh.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
