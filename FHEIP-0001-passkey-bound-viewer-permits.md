---
fheip: 0001
title: Passkey-bound viewer permits for gasless off-chain reveal
description: Register `p256-v1` as a first-class auth scheme in the FHEIP-0005 registry so a WebAuthn / P-256 passkey can authorize threshold-network decryption without a redundant secp256k1 keypair or an on-chain funding step per viewer rotation.
author: 0xOucan (@0xOucan)
discussions-to: TBD
status: Draft
type: Standards Track
category: Interface
created: 2026-04-16
requires: FHE.sol (cofhe-contracts), FHEIP-0005 (TN wire format v2), FHEIP-0006 (Permit V2)
---

## Abstract

`FHE.allow(ciphertext, viewer)` grants a 20-byte address the right to request threshold-network decryption of a ciphertext handle. Today the viewer is implicitly a secp256k1 EOA: the decryption request is authorized by an ECDSA signature recovered to `viewer`. For applications whose user identity is a WebAuthn / passkey credential (P-256), this forces a redundant secp256k1 key purely to satisfy the authorization path, plus an on-chain fund-and-pay step per viewer rotation.

This FHEIP registers `p256-v1` in the [FHEIP-0005](./FHEIP-0005-threshold-network-wire-format-v2.md) auth-scheme registry and defines the canonical passkey-derived viewer-address function, the request auth payload, and the corresponding `authData` encoding for [FHEIP-0006](./FHEIP-0006-permit-v2-rotation-batch-pluggable-auth.md) permits. No Solidity changes are required — `FHE.allow` continues to take a 20-byte address; the address just happens to be derived from a P-256 public key via a documented scheme.

## Motivation

A passkey-rooted wallet (e.g., Z0tz V6.5) uses one WebAuthn credential as the user's master identity. Today, to get a threshold decryption, such a wallet has to:

1. HKDF-derive a secondary secp256k1 private key from the passkey
2. Compute `viewerAddress = keccak(secp256k1_pubkey)[12:]`
3. Call `FHE.allow(handle, viewerAddress)` on-chain
4. Sign the decryption request with the derived secp256k1 key
5. Send it to the threshold network

The secondary secp256k1 key exists only to produce an ECDSA signature the network recognizes. The passkey is already a cryptographically strong identity; introducing a second keypair per viewer widens the key-material surface, forces the wallet to store / sync / rotate extra state, and rules out hardware passkeys where the raw private key is not exportable (e.g., platform authenticators on iOS / Android).

Registering P-256 as a first-class auth scheme collapses steps 1-2 to a deterministic derivation and lets step 4 sign natively with the passkey. Platform passkeys that never expose a raw private key become usable as FHE viewers without compromise.

### Use cases

- Passkey-native wallets (Z0tz, any ERC-4337 account whose owner is a P-256 pubkey)
- Mobile dapps with OS-provided passkeys (Apple Passkeys, Android Credential Manager, WebAuthn on mobile browsers)
- Multi-device sync where the passkey roams via cloud-backed credential managers but no secp256k1 derivations ever leave the device
- Hardware-bound identities (YubiKey, Secure Enclave) where the passkey private key is non-extractable

### Non-goals

- Changing on-chain `FHE.allow(handle, address)`. This proposal keeps the address-based ACL and maps pubkeys to deterministic addresses.
- Replacing secp256k1 on-chain signatures (smart-account owner, ERC-4337 UserOp sig). This FHEIP governs off-chain threshold-network authorization only.
- Defining how the passkey is stored / synced / recovered. That's the wallet's concern.

## Specification

### 1. Viewer-address derivation from a P-256 public key

Define the canonical mapping from a P-256 public key `(Qx, Qy)` to a 20-byte viewer address:

```
viewerAddress = keccak256(abi.encode(Qx, Qy, "CoFHE.P256Viewer/1"))[12:]
```

- `Qx`, `Qy` are 32-byte big-endian field elements.
- `"CoFHE.P256Viewer/1"` is the reserved UTF-8 domain tag for this derivation. It prevents collisions with addresses derived via other passkey schemes (smart-account factories, stealth-address derivations).
- The derivation is deterministic: the same `(Qx, Qy)` always produces the same address.

SDK API:

```typescript
// @cofhe/sdk/permits/p256
export function deriveP256ViewerAddress(Qx: Hex, Qy: Hex): Address;
```

### 2. `p256-v1` entry in the FHEIP-0005 auth-scheme registry

Registry row:

| Scheme ID | Signer | Auth payload |
|---|---|---|
| `p256-v1` | P-256 passkey (WebAuthn or raw) | `{ viewer: Hex, pubX: Hex, pubY: Hex, signature: Hex }` |

Canonical hash input — the P-256 signature is produced over:

```
msg      = sha256( canonicalJSON({ ctHash, viewer, chainId, deadline, requestId }) )
signature = P256.sign(privkey, msg)    // lowS: true, prehash: false
```

Field rules:

- `canonicalJSON(x)` produces deterministic JSON: keys sorted lexicographically, no whitespace, UTF-8 only, no trailing newline.
- `signature` is 64 bytes: `r` (32-byte big-endian) concatenated with `s` (32-byte big-endian). No `v` byte — P-256 decoders don't need it given the pubkey is explicit.
- `pubX`, `pubY` are 32-byte big-endian.

Threshold-network verification:

1. Re-derive `derivedViewer = deriveP256ViewerAddress(pubX, pubY)`.
2. If `derivedViewer != viewer` in the request → reject with `INVALID_SIGNATURE` (per FHEIP-0005 §4).
3. Verify the ACL grant: `FHE.allow(ctHash, viewer)` on `chainId` must be in effect.
4. Verify `P256.verify(pubX, pubY, msg, signature)` with `lowS: true`. Reject malleable signatures.
5. Check `deadline ≥ now - clock_skew` per FHEIP-0005.

On success, the network returns the FHEIP-0005 v2 response envelope.

### 3. `authData` encoding for FHEIP-0006 PermitV2

A PermitV2 authenticated by a P-256 passkey sets `authSchemeId = bytes32("p256-v1")` (UTF-8, left-padded with zeros) and encodes `authData` as:

```solidity
authData = abi.encode(
    uint256 pubX,        // P-256 public key X
    uint256 pubY,        // P-256 public key Y
    uint256 sigR,        // P-256 signature R
    uint256 sigS         // P-256 signature S (lowS)
);
```

The hash input for the P-256 signature, when authenticating a PermitV2, is:

```
msg = sha256( abi.encode(
    "CoFHE.P256Viewer/1",
    permitTypeHash,
    rotationCounter,
    issuer,
    keccak256(abi.encodePacked(ctHashes)),
    keccak256(abi.encodePacked(consumers)),
    deadline,
    chainId
))
```

Where `permitTypeHash = keccak256("PermitV2(uint16 version,uint32 rotationCounter,address issuer,bytes32[] ctHashes,address[] consumers,uint256 deadline,bytes32 authSchemeId)")` — matching FHEIP-0006 §4.

### 4. WebAuthn integration

For wallets using WebAuthn (browser or mobile) rather than a raw P-256 key, the signing flow is:

1. SDK constructs `msg` as in §2 or §3.
2. SDK calls `navigator.credentials.get({ publicKey: { challenge: msg, ... } })`.
3. From the returned `AuthenticatorAssertionResponse`, SDK extracts `(r, s)` by parsing the DER-encoded signature and normalizing `s` to lowS.
4. SDK re-computes the WebAuthn-signed message: `authenticatorData || sha256(clientDataJSON)` — this is what the authenticator actually signed, not `msg` directly.

Because WebAuthn's signed payload includes `authenticatorData` and `clientDataJSON`, the threshold network verifying the signature MUST also receive and verify these wrappers when `p256-v1` is used via WebAuthn. A future FHEIP may register `p256-webauthn-v1` as a distinct scheme carrying the WebAuthn-specific fields; this FHEIP scopes `p256-v1` to raw P-256 signatures over `msg` directly. Wallets using WebAuthn today SHOULD extract the raw signature and verify locally that it covers the intended `msg` before submitting, treating WebAuthn as a signing oracle.

## Rationale

**Why map pubkey → address instead of changing `FHE.allow` to accept a pubkey?** Contract storage of a 64-byte pubkey is 2.5× the cost of a 20-byte address per permit, and existing permit-revocation patterns (and FHEIP-0004's `revokeAllow`) key on `address`. Mapping pubkey → deterministic address keeps the on-chain surface stable and lets passkey and EOA viewers coexist in the same `allowed` mapping with no code changes.

**Why the domain tag `"CoFHE.P256Viewer/1"`?** Matches the domain-separation pattern used by FHEIP-0007 (`"CoFHE.Bind/1"`) and the `-v<N>` suffix used by FHEIP-0005's registry. The previous draft of this FHEIP used `"z0tz-viewer/1"`; the tag is generalized now that the proposal targets all passkey wallets.

**Why `lowS: true, prehash: false`?** lowS defeats signature malleability (a well-known ECDSA footgun). `prehash: false` means the SDK is responsible for hashing `msg` to the canonical 32-byte input, avoiding silent divergence between SDKs that pre-hash and threshold networks that do not. Noble-curves 2.x defaults differ from 1.x here — pinning both flags prevents the class of silent-verify-failures observed in Z0tz's relayer auth.

**Why defer WebAuthn to a future `p256-webauthn-v1`?** WebAuthn wraps the challenge in `authenticatorData + clientDataJSON` before signing. Treating WebAuthn as a distinct scheme keeps `p256-v1` simple (raw signature over `msg`) and lets the WebAuthn variant add its wrapper handling without breaking raw-P-256 callers. Wallets that only have WebAuthn today can extract and normalize, or wait for the dedicated scheme.

## Backwards Compatibility

Fully additive. FHEIP-0005 v1 clients (`ethsign-v1`) continue to work unchanged. Threshold-network operators add a `p256-v1` verifier alongside the existing ECDSA verifier. The on-chain `FHE.allow` surface does not change. Wallets can register a P-256 viewer and an EOA viewer for the same handle simultaneously.

## Security Considerations

1. **Pubkey substitution.** The threshold network MUST re-derive `viewer` from `(pubX, pubY)` and reject if it doesn't match the `viewer` field in the request. Without this check, an attacker could present a victim's ACL-allowed address paired with the attacker's own pubkey and decrypt for free.
2. **Signature malleability.** P-256 ECDSA is malleable; `(r, s)` and `(r, -s mod n)` are both valid for the same message. Implementations MUST enforce `lowS: true` on both sign and verify. A future "malleable" signature submitted to the network MUST be rejected.
3. **`canonicalJSON` bit-exactness.** Any disagreement between signer and verifier on JSON canonicalization (whitespace, key ordering, escape rules) causes silent signature failure. Pin the canonicalization algorithm (e.g., RFC 8785 JCS) and test with shared fixtures.
4. **WebAuthn user-verification posture.** Wallets integrating WebAuthn SHOULD request `userVerification: "required"` so the authenticator forces biometric / PIN presence per decryption. This is stronger than the `ethsign-v1` path, which has no presence check at all.
5. **Viewer-address collisions.** The keccak-truncation inherits the same 2^160 collision space as Ethereum addresses. This is acceptable; contract semantics are unchanged. Two different `(Qx, Qy)` pairs colliding to the same viewer address is cryptographically improbable but not impossible; ACL semantics treat it as a single viewer.
6. **Key rotation.** Rotating a passkey-derived viewer is identical to rotating any viewer: `FHE.allow` the new derived address, `FHE.revokeAllow` (FHEIP-0004) the old. The passkey itself does not need to rotate. Device-loss rotation works out of the box if the user has another passkey enrolled on the same account.
7. **Deadline enforcement.** Requests with `deadline < now - clock_skew` MUST be rejected. FHEIP-0005 recommends `clock_skew = 300`. Wallets SHOULD set `deadline = now + 600` to leave comfortable margin for network latency.
8. **Cross-scheme viewer reuse.** Because the viewer address is derived with a domain tag, it will NOT collide with a smart-account factory's `CREATE2` address using the same pubkey or a stealth-address derivation. Consumers MUST preserve the domain tag verbatim and not "simplify" it.

## Reference Implementation

- Pubkey → viewer derivation prototype: [`cli/src/ledger/idDerivation.ts`](https://github.com/0xOucan/Z0tz/blob/main/cli/src/ledger/idDerivation.ts) (function `deriveViewerKey`, currently using secp256k1 under the hood but structured for a passkey-only swap).
- P-256 signing and lowS normalization: [`relayer/lib/auth.ts`](https://github.com/0xOucan/Z0tz/blob/main/relayer/lib/auth.ts) — the relayer's P-256 auth path exercises the same signature shape this FHEIP registers, including canonical-JSON prehash and lowS signing.
- The gasless-reveal flow that motivated this proposal: [`cli/src/ledger/reveal.ts`](https://github.com/0xOucan/Z0tz/blob/main/cli/src/ledger/reveal.ts) — shows how a viewer with `FHE.allow` access decrypts off-chain without funding an EOA.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
