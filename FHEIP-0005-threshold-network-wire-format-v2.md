---
fheip: 0005
title: Threshold-network wire format v2 with auth-scheme registry
description: Canonicalize the CoFHE threshold-decryption API as a versioned JSON envelope with a pluggable `authScheme` discriminator so secp256k1, P-256, EIP-712 permits, and future schemes coexist in one wire format with predictable evolution.
author: 0xOucan (@0xOucan)
discussions-to: TBD
status: Draft
type: Standards Track
category: Interface
created: 2026-04-17
requires: FHE.sol (cofhe-contracts), CoFHE threshold-decryption API, FHEIP-0001
---

## Abstract

The CoFHE SDK today ships two decryption endpoints (`/decrypt` and `/v2/decrypt`) with request and response shapes defined only by their TypeScript call sites. There is no version negotiation, no authentication-scheme registry, and no canonical error code set. Adding a new auth scheme (P-256 passkey in FHEIP-0001, multi-sig, delegated session keys) requires coordinated changes across the SDK, the threshold-network service, and every relayer. This FHEIP proposes a versioned JSON envelope with an `authScheme` discriminator pointing into a public registry, a canonical error-code set, and a simple HTTP header for version negotiation. Old clients continue to work via v1; new clients opt into v2 and gain the registry.

## Motivation

### The wire format is implicit

Today the only specification for the decrypt API is TypeScript client code. A threshold-network implementor reading that code has to reverse-engineer:

- Which HTTP verb / path is canonical?
- What fields are required vs optional?
- How is chainId conveyed when the handle is cross-chain?
- What shape does a signed request take under each auth method?
- How does the response encode errors vs successes?

Two clients have already diverged (`tnDecryptV1.ts` vs `tnDecryptV2.ts`) and neither is authoritative.

### The auth scheme is hard-coded

Every request assumes secp256k1 ECDSA over a fixed message. FHEIP-0001 introduces P-256 passkey permits; there's no slot to put the scheme id. New schemes (batch permits, delegated keys, multi-sig ACL) will each require bespoke endpoints.

### Errors are strings

`error_message` is free-form text. Clients that want to retry on transient errors vs surface permission failures have to string-match, which breaks on server-side text changes.

### Use cases

- Threshold-network operators running independent implementations that must interop
- SDK authors building multi-scheme permit flows
- Indexers / audit tools that need to replay decryption attempts deterministically
- Bridges between different FHE networks

### Non-goals

- Replacing on-chain ACL (`FHE.allow`). This is a wire-format spec for off-chain authorization only.
- Defining new encryption primitives.
- Mandating a specific transport (JSON over HTTPS is normative, gRPC / Protobuf may be offered alongside but must map 1:1 to the JSON shape).

## Specification

### 1. Request envelope v2

```
POST /v2/decrypt HTTP/1.1
Host: <threshold-network-host>
Content-Type: application/json
X-CoFHE-API-Version: 2
```

Body:

```json
{
  "version": 2,
  "requestId": "<UUIDv4>",
  "chainId": 84532,
  "ctHash": "0x<32-byte hex>",
  "authScheme": "<registry-id>",
  "auth": { "...scheme-specific...": "..." },
  "deadline": 1729200000
}
```

Field semantics:

- `version`: integer. MUST equal 2. Mismatch → HTTP 400 with `INVALID_VERSION`.
- `requestId`: client-generated UUIDv4. The TN MAY dedupe on `(requestId, client-ip)` for idempotency.
- `chainId`: EIP-155 chain ID hosting the `FHE.allow` grant.
- `ctHash`: 32-byte ciphertext handle.
- `authScheme`: registry identifier (see §3). String, ≤32 bytes.
- `auth`: opaque object whose shape is defined by the scheme.
- `deadline`: unix seconds. Requests with `deadline < now - clock_skew` MUST be rejected with `EXPIRED`. Recommended `clock_skew = 300`.

### 2. Response envelope v2

```json
{
  "version": 2,
  "requestId": "<echoes request>",
  "status": "ok",
  "decrypted": "0x<hex>",
  "encoding": "uint64-le",
  "signature": "0x<r||s||v hex>"
}
```

or on failure:

```json
{
  "version": 2,
  "requestId": "<echoes request>",
  "status": "error",
  "errorCode": "<registry-id>",
  "errorMessage": "<human-readable>"
}
```

Field semantics:

- `status`: MUST be `"ok"` or `"error"`.
- `decrypted`: raw bytes of the plaintext, hex-encoded. MUST be absent when status is `"error"`.
- `encoding`: one of `"uint8-le"`, `"uint16-le"`, `"uint32-le"`, `"uint64-le"`, `"uint128-le"`, `"bool"`, `"address"`. Little-endian for unsigned ints to match EVM word layout. Clients MUST check `encoding` and MUST NOT hard-code the decryption handle's type.
- `signature`: threshold-network attestation over `keccak256(abi.encode(ctHash, decrypted, chainId, deadline))`. Signing curve depends on network config; see §5.
- `errorCode`: from the error registry (§4). Stable identifier clients can switch on.

### 3. `authScheme` registry (initial entries)

| Scheme ID | Signer | Auth payload |
|---|---|---|
| `ethsign-v1` | secp256k1 EOA | `{ signer: 0x<address>, signature: 0x<r\|\|s\|\|v> }` over EIP-191 `"\x19Ethereum Signed Message:\n32" + keccak256(abi.encode(ctHash, chainId, deadline))` |
| `permit-eip712-v1` | secp256k1 EOA | `{ signer: 0x<address>, signature: 0x<r\|\|s\|\|v>, permit: <Permit struct> }` — full EIP-712 over the permit domain |
| `p256-v1` (FHEIP-0001) | P-256 passkey | `{ viewer: 0x<derived-address>, pubX, pubY, signature: 0x<r\|\|s> }` over sha256(canonical JSON of `{ctHash, viewer, deadline, requestId}`) |
| `erc1271-v1` | smart contract | `{ signer: 0x<contract-address>, signature: 0x<bytes> }` — verified by calling `isValidSignature` at the signer |

New entries added by follow-up FHEIPs. Scheme IDs are lowercase, kebab-case, suffixed with `-v<N>` for versioning. IANA-style registration not in scope; a `registry.json` in this repo will track entries.

### 4. Error-code registry

| Code | Meaning | Retry? |
|---|---|---|
| `INVALID_VERSION` | `version` field unrecognized | no |
| `INVALID_CTHASH` | ciphertext handle unknown on `chainId` | no |
| `UNAUTHORIZED` | viewer not in ACL for this handle | no |
| `EXPIRED` | `deadline` past | no |
| `INVALID_SIGNATURE` | auth payload does not verify | no |
| `UNSUPPORTED_SCHEME` | `authScheme` not in registry | no |
| `RATE_LIMITED` | per-client quota exceeded | yes, with backoff |
| `TEMPORARILY_UNAVAILABLE` | network partition or quorum loss | yes, with backoff |
| `INTERNAL_ERROR` | unexpected server state | yes once |

### 5. Version negotiation

- `GET /health` MUST return `{ "versions": [1, 2, ...] }` so clients can pick.
- Requests without `X-CoFHE-API-Version` default to v1 for backwards compatibility. v2 clients MUST send the header explicitly.
- Responses carry `X-CoFHE-API-Version` matching the served version.

### 6. Response signature curves

The `signature` field is produced by the threshold network; curve is fixed per deployment and exposed via `GET /config`:

```json
{ "responseSigCurve": "secp256k1-ethereum", "signerAddress": "0x..." }
```

Alternate curves (e.g., BLS aggregate) may be introduced in a later FHEIP; clients MUST consult `/config` rather than assume secp256k1.

## Rationale

**Why JSON over Protobuf?** The existing SDK is TypeScript; JSON is zero-friction. A Protobuf variant may be offered for gRPC clients but must mirror the JSON 1:1 (same field names, same error codes).

**Why a string `authScheme` and not an integer?** Strings are self-documenting in logs and let registries grow without central coordination. 32-byte cap keeps them cheap to index.

**Why EIP-191 for `ethsign-v1` and not raw keccak?** EIP-191 is the existing canonical envelope for secp256k1-signed off-chain messages; using it means any wallet supporting `personal_sign` can sign decrypt requests without custom tooling.

**Why `requestId` client-generated?** It lets clients correlate retries and the TN dedupe by `(client, requestId)` without maintaining server-side ordering.

**Why separate `decrypted` and `encoding` fields?** `decrypted` is a raw byte string; `encoding` tells the client how to interpret it. This avoids per-type endpoints.

## Backwards Compatibility

v1 clients continue to work unchanged — the TN MUST keep serving `/decrypt` alongside `/v2/decrypt` for a deprecation window. v2-only features (new auth schemes, new error codes) are unavailable on v1 by design. The registry in §3 MAY back-port scheme IDs to v1 as a best-effort compatibility layer.

## Security Considerations

1. **Request replay.** `deadline` + per-request `requestId` bound replay within the clock-skew window. Clients SHOULD use fresh UUIDs per request; TNs SHOULD dedupe on `(requestId, viewer)` within a TTL window matching `deadline + skew`.
2. **Pubkey substitution in `p256-v1`.** See FHEIP-0001 §5.1 — the TN MUST re-derive the viewer address from `(pubX, pubY)` and reject mismatches.
3. **Scheme-specific field tampering.** Each scheme ID's canonical hash input MUST include every security-critical field (ctHash, chainId, deadline, requestId at minimum). Schemes that omit any of these are unsafe and MUST NOT be registered.
4. **Error-code leakage.** `UNAUTHORIZED` vs `INVALID_CTHASH` reveal whether a handle exists. This is the same exposure as `FHE.allow` on-chain; dapps that need to hide existence must use constant-time client behavior.
5. **Response signature verification.** Clients MUST verify the response signature before trusting `decrypted`. An unsigned response — even over TLS — is malleable if the TN is compromised.
6. **Version downgrade.** A MITM could strip `X-CoFHE-API-Version` to force v1 semantics. Clients MUST pin the minimum version they accept and reject older responses.
7. **`erc1271-v1` reentrancy.** Verifying `isValidSignature` calls into the signer contract. TNs MUST bound gas for this verification call and treat any revert as `INVALID_SIGNATURE`.

## Reference Implementation

- Current v1/v2 divergent clients: `cofhesdk/packages/sdk/core/decrypt/tnDecryptV1.ts` — function `tnDecryptV1` at line 68 with an inline `{ ct_tempkey, host_chain_id, permit? }` request body at lines 74-81; `tnDecryptV2.ts` — `submitDecryptRequestV2` at line 105 with the same inline shape at lines 111-118. Neither exports a request/response type; both are inlined at the call site.
- Z0tz's existing P-256 signing for relayer auth demonstrates the signature shape that `p256-v1` would upstream: [`relayer/lib/auth.ts`](https://github.com/0xOucan/Z0tz/blob/main/relayer/lib/auth.ts) — canonical-JSON body prehash with sorted keys, SHA-256, `lowS: true`, 64-byte `r||s` output.
- The `requestId` + `deadline` pattern is already deployed in Z0tz's relayer protocol and has caught both client-side clock drift and replay attempts during integration testing.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
