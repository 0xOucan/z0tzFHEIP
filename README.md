# FHEIP — Fhenix Improvement Proposals (Z0tz drafts)

Proposals drafted by the Z0tz team for upstream contribution to the CoFHE SDK, `FHE.sol`, `FHERC-20`, and the threshold-decryption protocol. Format follows [EIP-1](https://eips.ethereum.org/EIPS/eip-1), with specification conventions borrowed from [EIP-191](https://eips.ethereum.org/EIPS/eip-191) (byte-layout precision) and [EIP-1271](https://eips.ethereum.org/EIPS/eip-1271) (interface-first spec style).

These drafts are **pre-discussion** — they've been scoped against real implementation experience in Z0tz (passkey-rooted wallet, FHE-encrypted ledger, cross-chain CCTP bridging) but have not yet been filed to an upstream tracker. Use them as a starting point for conversations with @FhenixIO / the cofhe-sdk maintainers.

| # | Title | Category | Status |
|---|---|---|---|
| 0001 | [Passkey-bound viewer permits for gasless off-chain reveal](./FHEIP-0001-passkey-bound-viewer-permits.md) | SDK | Draft |
| 0002 | [Solvency-checked FHE debit primitive](./FHEIP-0002-solvency-checked-fhe-debit.md) | Core | Draft |
| 0003 | [Confidential ledger event schema](./FHEIP-0003-confidential-ledger-event-schema.md) | ERC | Draft |
| 0004 | [ACL lifecycle — transient scope, persistent revocation, cross-call forwarding](./FHEIP-0004-acl-lifecycle-spec.md) | Core | Draft |
| 0005 | [Threshold-network wire format v2 with auth-scheme registry](./FHEIP-0005-threshold-network-wire-format-v2.md) | Interface | Draft |
| 0006 | [Permit V2 — rotation, batch, pluggable auth-scheme](./FHEIP-0006-permit-v2-rotation-batch-pluggable-auth.md) | SDK | Draft |
| 0007 | [Ciphertext integrity binding in signed digests](./FHEIP-0007-ciphertext-integrity-binding.md) | Core | Draft |
| 0008 | [FHERC-20 wrapper decimal-alignment standard](./FHEIP-0008-fherc20-wrapper-decimal-alignment.md) | ERC | Draft |
| 0009 | [Canonical self-call binding helper for relayer-submitted FHE ops](./FHEIP-0009-fhe-self-call-binding-helper.md) | Core | Draft |

Naming: `FHEIP-XXXX-short-kebab-title.md`, 4-digit zero-padded, incrementing.

## Relationships between the drafts

- **0005** defines the auth-scheme registry; **0001** (P-256), **0006** (Permit V2 auth slot), and future schemes plug into it.
- **0006** depends on **0005** for the scheme discriminator format.
- **0004** (ACL lifecycle) and **0009** (self-call helper) are complementary — 0004 defines *when* ACL grants expire and how to forward them, 0009 defines *who* the grant binds to when inputs come through a relayer.
- **0007** (ciphertext binding) and **0009** (self-call) together defeat the full amount-substitution attack class: 0007 prevents the relayer from swapping the ciphertext, 0009 prevents the relayer from becoming the ACL holder.
- **0002** (solvency primitive) and **0003** (event schema) together give any confidential-ledger dapp a full "safe debit + legible history" stack. The two events are intentionally disjoint: 0003's `ConfidentialDebit(accountKey, destination, encAmount, op)` is the indexer primitive for flow reconstruction; 0002's `ConfidentialDebitResult(accountKey, encSuccessFlag, encAmount)` is the viewer primitive for solvency introspection. A dapp can emit both in the same tx without collision.
- **0008** (wrapper decimals) references **0003** for the event-emission pattern.

## Companion research documents

Alongside the FHEIPs, this repo carries three research-grade pieces that ground the proposals in a real consumer dapp (Z0tz V6.5) and compare its approach to adjacent designs in the space.

| Document | Purpose |
|---|---|
| [Z0tz V6.5 article](./z0tz-v6.5-article.md) | Full architectural writeup of Z0tz V6.5, structured after the Fhenix *Fluton × Fhenix* announcement — confidentiality + anonymity via FHE, stealth, pseudonymous ledger, and CCTP composition. |
| [Z0tz V6.5 essay](./z0tz-v6.5-essay.md) | Narrative essay on the V6.5 design: "Privacy is a stack, not a feature" — long-form prose on the composition bet that ties FHE, stealth, pooled vault, and permissionless bridges into one wallet. |
| [Z0tz vs Fluton vs UTXO comparison](./z0tz-vs-fluton-utxo-comparison.md) | Research comparison across three approaches to confidential ledgers — Fluton (FHE + smart-account anonymity + solver routing), UTXO confidential tokens (ZK commitments with optional wormhole), and Z0tz V6.5. Maps the design space without ranking. |
