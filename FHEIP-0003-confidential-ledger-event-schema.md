---
fheip: 0003
title: Confidential ledger event schema
description: Define a canonical event family — `ConfidentialTransfer`, `ConfidentialMint`, `ConfidentialBurn`, `ConfidentialCredit`, `AccountRotated` — that any FHE-accounting contract SHOULD emit so indexers and wallets can rebuild confidential balances across dapps without per-contract custom parsing.
author: 0xOucan (@0xOucan)
discussions-to: TBD
status: Draft
type: Standards Track
category: ERC
created: 2026-04-17
requires: FHERC-20 (fhenix-confidential-contracts), FHE.sol (cofhe-contracts)
---

## Abstract

FHERC-20 today emits only `Transfer(from, to, indicatorTick)` — an opaque 0-9999 bucket that carries no ciphertext handle. Every confidential-ledger dapp therefore invents its own event shape (`TransferredOut(address, bytes32)`, `CreditedFromVault(bytes32, uint256)`, `Spent(bytes32 oldId, bytes32 newId, uint8 action)`, and so on). The result is that no two confidential ledgers share an indexer, block explorer, or history-scanner implementation. This FHEIP proposes a minimal canonical event family that any FHE-accounting contract SHOULD emit for credits, debits, mints, burns, and account-identity rotations. Indexers that understand these five events can reconstruct balance histories, rotation chains, and vault flows across any compliant contract without dapp-specific logic.

## Motivation

Reconstructing a confidential-ledger history from chain state is only possible when events expose:

1. **Which account key changed.** Indexed so scanners can filter.
2. **What ciphertext handle is the new/delta value.** So a viewer can optionally decrypt.
3. **Where it came from / went to.** For cross-contract flows (vault → ledger, burn → mint).

FHERC-20 exposes #1 weakly (the `indicatorTick` is not tied to any particular operation), nothing for #2, and nothing for #3. Real-world consumers hit this wall immediately:

- A "rebuild my balance history" wallet needs per-contract event ABIs.
- A block explorer cannot render "you received 1.23 tokens (encrypted)" without the ciphertext handle in an indexed slot.
- Scanners debugging cross-chain flows (CCTP burn → mint into encrypted ledger) must correlate by tx hash + fragile source address matching. Z0tz's CCTP history scanner spent three revisions chasing event-ABI mismatches (`burnToken` is `address` not `bytes32`; `encAmount` is `bytes32` not `uint256`) that would not have happened with a standard.

### Use cases

- Confidential-ledger wallets (scan chain → rebuild balance + history)
- Block explorers (render encrypted-balance activity without per-dapp code)
- Compliance / audit tooling (decrypt history with a viewer permit, same shape for every dapp)
- Cross-chain scanners stitching mint/burn/credit events across CCTP-like bridges

### Non-goals

- Mandating ciphertext-handle format (that's the FHE.sol type system's job).
- Replacing `Transfer(from, to, indicatorTick)`. FHERC-20 keeps indicator semantics for ERC-20 compatibility; this FHEIP is additive.
- Defining cross-chain message framing (covered by existing bridge standards).

## Specification

### 1. Core event set

```solidity
/// Credit to an account's encrypted balance (mint, inbound transfer, vault credit).
event ConfidentialCredit(
    bytes32 indexed accountKey,
    bytes32 indexed source,     // bytes32(0) for pure mint; else source account or contract
    bytes32          encAmount,  // euint64/128 ciphertext handle
    uint8            op          // 0=mint, 1=transfer-in, 2=vault-credit, 3=bridge-mint
);

/// Debit from an account's encrypted balance (burn, outbound transfer, vault debit).
event ConfidentialDebit(
    bytes32 indexed accountKey,
    bytes32 indexed destination, // bytes32(0) for pure burn; else dest account or contract
    bytes32          encAmount,  // euint64/128 handle
    uint8            op          // 0=burn, 1=transfer-out, 2=vault-debit, 3=bridge-burn
);

/// Account identity rotation (new key, same owner). Used by passkey-rooted ledgers.
event AccountRotated(
    bytes32 indexed oldAccountKey,
    bytes32 indexed newAccountKey
);

/// First-time registration of an account key.
event AccountRegistered(
    bytes32 indexed accountKey,
    bytes32          ownerHash,   // e.g. keccak(pubkey) — identity the key binds to
    address          viewer        // address holding decryption ACL
);

/// Account key deletion (funds rotated away, registration closed).
event AccountClosed(
    bytes32 indexed accountKey
);
```

### 2. Indexing rules

- `accountKey`, `source`, `destination`, `oldAccountKey`, `newAccountKey` are indexed. Scanners MUST be able to filter by account without decryption.
- Ciphertext handles (`encAmount`) are non-indexed. Indexing them would burn gas without benefit: decryption requires a viewer permit, not a log topic match.
- `op` is a compact 8-bit discriminator inside the event, not a separate event per op. Keeping the event shape uniform means one ABI entry covers every credit path; the discriminator is cheaper than a topic and avoids log-count explosion.

### 3. ERC-20 compatibility bridge

Contracts implementing both FHERC-20 and this FHEIP MUST emit both `Transfer(from, to, indicatorTick)` (ERC-20) and `ConfidentialCredit` / `ConfidentialDebit` (this FHEIP) for every balance-changing operation. The indicator-tick semantics are unchanged; this FHEIP adds information, it does not replace the existing event.

### 4. Vault / cross-contract flows

When balance motion crosses contracts (vault → ledger, wrapper → underlying), both contracts SHOULD emit matching events:

- Source emits `ConfidentialDebit(accountKey=srcAccount, destination=bytes32(uint256(uint160(destContract))), encAmount, op=2)`.
- Destination emits `ConfidentialCredit(accountKey=dstAccount, source=bytes32(uint256(uint160(srcContract))), encAmount, op=2)`.

The `encAmount` handle MAY differ between source and destination — ACL forwarding typically allocates a fresh handle on the destination side. Scanners correlate by tx hash, not by handle equality.

### 5. Relationship to `ConfidentialDebitResult` (FHEIP-0002)

`ConfidentialDebit` (this FHEIP) logs that a debit occurred with source / destination / op metadata. `ConfidentialDebitResult` (FHEIP-0002) separately logs the solvency-gate outcome as an `ebool` ciphertext decryptable only by viewers with the right ACL. A contract supporting both MAY emit them in the same tx: one `ConfidentialDebit` per debit for indexer flow reconstruction, and one `ConfidentialDebitResult` per debit for viewer-side success introspection. Clients that only need the flow graph ignore `ConfidentialDebitResult`; clients that only need solvency outcomes ignore `ConfidentialDebit`.

### 6. Bridge semantics

For cross-chain flows (CCTP, LayerZero, Hyperlane):

- Source chain: `ConfidentialDebit(accountKey, destination=domain-id, encAmount, op=3)`.
- Dest chain: `ConfidentialCredit(accountKey, source=domain-id, encAmount, op=3)`.

Where `destination`/`source` encode the CCTP domain ID (or equivalent) in the low bits. This lets a cross-chain scanner stitch without parsing the bridge's internal payload.

## Rationale

**Why two events, not one `ConfidentialBalanceChange(delta)` event?** FHE cannot express signed deltas naturally (`euint64` is unsigned); a debit event with a separate "direction" flag is simpler than a signed-delta handle. Splitting also lets indexers subscribe to credits only or debits only when building read-oriented UIs.

**Why a `uint8 op` discriminator instead of per-op events?** Uniform shape lets indexer ABIs stay small. Event count per block matters for log storage / RPC cost; a single event type with 4 discriminator values halves log-table cardinality compared to `ConfidentialMint` / `ConfidentialTransferIn` / `ConfidentialVaultCredit` / `ConfidentialBridgeMint` as four distinct events.

**Why `bytes32` instead of `address` for account keys?** Ledger IDs, vault slots, and other custom identifiers don't fit 20 bytes. Implementations keyed on EOAs pad the address; nothing is lost.

**Why `ownerHash` in `AccountRegistered` and not the pubkey directly?** A 32-byte hash is cheaper (one indexed slot vs two) and matches the pattern FHERC-20 wrappers already use (`keccak(pubkey)`). Wallets that need the full pubkey can still record it in a separate event.

## Backwards Compatibility

Additive. FHERC-20 contracts that emit only `Transfer(indicatorTick)` remain compliant with FHERC-20 — this FHEIP adds events, does not remove them. Indexers unaware of `ConfidentialCredit` / `ConfidentialDebit` skip them. A contract can opt in incrementally (emit the new events alongside old ones) without breaking existing callers.

## Security Considerations

1. **Ciphertext handle exposure.** The handle in `encAmount` is not secret — handles are public on-chain references. Decryption requires `FHE.allow(handle, viewer)`. Emitting a handle in a log does not grant anyone the right to decrypt it.
2. **Correlation via `accountKey`.** Indexed account keys let anyone scan "all activity on this account." This is the same exposure as ERC-20 `Transfer(from, to)`. Privacy-maximizing dapps (e.g., Z0tz cash-in stealths) already use one-shot account keys to avoid this.
3. **Cross-contract spoofing.** A contract emitting `ConfidentialCredit(source=<victim>)` cannot actually credit anyone — events don't move balances. But a naive scanner that believes events without checking the emitting contract could attribute funds incorrectly. Scanners MUST verify the emitting contract is trusted before crediting balance in their reconstruction.
4. **Op discriminator drift.** Implementations MUST NOT invent local op codes outside {0,1,2,3}. Extending the registry requires a follow-up FHEIP.

## Reference Implementation

- Z0tz's non-standard confidential-ledger events: [`Z0tzPrivateLedger.sol` lines 90-93](https://github.com/0xOucan/Z0tz/blob/main/contracts/contracts/ledger/Z0tzPrivateLedger.sol#L90-L93) (`Registered`, `CreditedFromVault`, `Spent`, `Rotated`) — plus `TransferredOut(address,bytes32 encAmount)` on the companion vault contract, which scanners must discover separately.
- A chain-rebuilt history scanner works around the lack of a canonical event schema: [`gui/src/main/history-scanner.ts`](https://github.com/0xOucan/Z0tz/blob/main/gui/src/main/history-scanner.ts) reconstructs five canonical confidential-ledger flows (same-chain cash in / out, cross-chain cash in / out, self-bridge) by correlating tx hashes across six event shapes on four contracts and two ABI quirks that caused real bugs (`DepositForBurn.burnToken` is `address` not `bytes32`; `TransferredOut.encAmount` is `bytes32` not `uint256`). With this FHEIP most of the scanner's per-contract logic collapses to one ABI.
- FHERC-20 current `Transfer(indicatorTick)` event: `fhenix-confidential-contracts/contracts/FHERC20.sol` in `_update` around lines 395-444.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
