# Z0tz V6.5 — Confidentiality meets anonymity via FHE, stealth, and passkey-pseudonymous accounting

*Structured after the Fluton × Fhenix announcement. Same dual-privacy problem, different primitives, running on three testnets today.*

## Context: Why This Matters

### Quick Primer: What FHE Accomplishes

Fully Homomorphic Encryption lets a smart contract compute on encrypted inputs and return encrypted outputs. On Fhenix CoFHE, a contract declares parameters as `euint8…euint128`, consumes `InEuintN` ciphertexts from the client, and runs arithmetic, boolean logic, and address ops entirely under encryption. Plaintext only reappears at the wallet, via a threshold-network decryption authorized by a permit.

This is a *confidentiality* primitive. It hides values. It does not hide **who** is computing.

### Anonymity vs. Confidentiality

- **Confidentiality** hides *what* — amounts, tokens, strategies. Observers see ciphertext.
- **Anonymity** hides *who* — the identity behind the action. Observers cannot link activity to a persistent wallet.

A wallet with perfect confidentiality but no anonymity leaks a graph: same address, same cadence, same counterparty set — identity narrows even without the amounts. A wallet with perfect anonymity but no confidentiality leaks values: a known amount lands at a fresh address, and any exchange deposit stream reveals the origin. Real privacy requires both.

### The Gap

Ethereum's accounting model was built for transparency. Every address is a permanent identifier. Every balance is a public mapping. FHE encrypts the values in events but not the addresses. Before V6.5, Z0tz used stealth addresses at the edges, but the user's FHERC-20 balance still lived in a mapping keyed by their smart-account address. The amount was encrypted; the holder was not. V6.5 closes that last surface.

## Z0tz's Design Choices

### a. Confidentiality via FHE

Every balance and every in-wallet transfer is a Fhenix CoFHE `euint64`. `FHERC20WrappedERC20` handles shield / unshield; `Z0tzPrivateLedger` keeps per-user encrypted entries; `Z0tzPrivateLedgerVault` holds the pooled FHERC-20. Solvency is checked via `FHE.lte + FHE.select` — on insolvency the contract silently transfers 0 rather than reveal the comparison (FHEIP-0002 canonicalizes this primitive).

### b. Anonymity via stealth + sweeper mixing + passkey-pseudonymous ledger

Z0tz takes a different route from "fresh smart account per user." The user has **one** smart account for recovery and CCTP signing, but it never appears in confidential-token state. Four mutually reinforcing mechanisms close the identity surface:

- **Incoming payments** route through one-time stealth addresses (ERC-5564 / 6538) derived per-transaction from the recipient's meta-address.
- **Sweeper mixing without a pool.** `Z0tzPrivateSweeperV2` is the `msg.sender` for every shield operation across the entire user population. An observer watching the FHERC-20 and vault events sees a uniform stream of `sweeper → vault` operations that look identical regardless of which user originated each one. No queue, no exit delay, no separate mixing protocol — the mixing is a side effect of routing.
- **Per-user balances** live under `ledgerId = HKDF(passkey, "z0tz-ledger-id", vault, nonce)` — not derived from any Ethereum address. Re-derivable on any device with the passkey; unlinkable on-chain.
- **Outgoing payments** route through another one-time stealth: ledger debits, vault sends encrypted FHERC-20 to the stealth, stealth unshields, stealth forwards to target.
- **Automatic per-spend rotation.** Every `spend` op carries an optional `newId`. The ledger debits the old, credits change to the new, deletes the old — atomic, ~25K gas. Intra-ledger clustering collapses to one op.

### c. Relayer / Paymaster

`Z0tzPaymaster` sponsors every user op and collects 1% in the transacted token at cash-in. Users hold zero ETH, reveal zero gas patterns.

The relayer is a thin HTTP service authenticated per request with a P-256 passkey signature. Benign-but-not-trusted threat model: the relayer can censor or delay but cannot steal funds, forge signatures, or substitute ciphertexts — the spend digest binds `op.amount.ctHash` (FHEIP-0007).

### d. CCTP composition — the composability proof

Z0tz doesn't operate the bridge. Circle does, via permissionless CCTP V2 at known addresses on every supported chain. Z0tz routes the burn-and-mint pair through a stealth pair, and the destination-chain sweeper post-mixes the mint back into the user's ledger. CCTP's public events name stealths on both ends, never the user.

Three flows, identical privacy properties:
- **Bridge** — encrypted on A → encrypted on B.
- **Cross-chain cashout** — encrypted on A → plaintext target on B.
- **Cross-chain cash-in** — plaintext stealth on A → encrypted ledger on B.

The reason this is worth highlighting beyond "cross-chain bridge" is that CCTP is the hardest composition case: two chains running simultaneously, an unavoidable plaintext window at the bridge boundary, and no privacy-friendly alternative in production. The fact that the pattern works for CCTP is evidence that the same three-part template (pre-stage at stealth → interact with external protocol → post-mix through sweeper) generalizes to any permissionless on-chain protocol — DEXes, lending markets, NFT mints, governance, airdrop claims. The external protocol doesn't need to know Z0tz exists; privacy travels with the user through whatever they choose to interact with.

## Deep Dive: How Z0tz Achieves Anonymity + Confidentiality

### A. Rationale

No single primitive covers both dimensions. FHE hides values but not identity. Stealth hides single transactions but leaks cadence on reuse. Pooled vaults close the per-user holder leak but still index per-user balances if the ledger keys on addresses. Pseudonymous ledger keys close the mapping-key leak but still cluster on reuse. Each layer exists because the previous layer leaks something. Composition, not replacement.

### B. Integration

```
External EOA (plaintext)
       │ ERC-20 to one-time stealth
       ▼
Stealth  ── stealth-signed sweep ──► Sweeper ──1%──► Treasury
                                        │
                                        │ shield + credit
                                        ▼
                                 Vault (pooled FHERC-20)
                                        │
                                        │ creditFromVault(ledgerId, pubkeyHash)
                                        ▼
                         Ledger.entries[ledgerId]  ← encrypted balance
```

Events involving the user name only: a one-time stealth, the shared sweeper (the same `msg.sender` for every user's shield), a pooled vault (the same holder for every user's balance), or a passkey-derived ledgerId. The smart account is absent.

Outgoing reverses: `ledger.spend({action: Cashout, destAddress: stealth})` → vault sends encrypted FHERC-20 to stealth → stealth unshields via threshold network → stealth forwards plaintext to target.

### C. Building Blocks

- **Z0tzPaymaster** — ERC-4337 paymaster, 1% token fee.
- **Z0tzPrivateSweeperV2** — stealth → ledger in one tx via `privateSweepToLedger`.
- **Z0tzPrivateLedgerVault** — pooled FHERC-20 holder, ~130 lines, no admin path post-lock.
- **Z0tzPrivateLedger** — P-256 auth via RIP-7212, atomic debit + rotation + (internal | cashout) in one `spend`.
- **RecoveryModule** — guardian-based recovery, lives on the smart account.
- **StealthAnnouncer / StealthAddressRegistry** — ERC-5564 / 6538.
- **CCTP wrapper** — composition glue around `TokenMessengerV2.depositForBurn` and `MessageTransmitterV2.receiveMessage`.
- **Passkey library** — P-256 on the client, RIP-7212 on-chain, ~3.5K gas/verify.

### D. Impact

The user's persistent identity does not appear in:
- any FHERC-20 state mapping
- any FHERC-20 event indexed topic
- any ledger event at cash-in time (the ledgerId is the key; it's a pseudonym)
- any CCTP burn / mint event at the bridge boundary

The only identity surface is the ledgerId, and it rotates on every spend.

Measured gas, Base Sepolia 5 gwei, April 2026:

| Flow | Gas | L2 cost |
|---|---:|---:|
| Cash-in | 591 K | $0.012 |
| Internal transfer + rotation | 405 K | $0.008 |
| Same-chain cashout (pre-unshield) | 683 K | $0.014 |
| Cross-chain bridge via CCTP | 1.5 M | $0.030 |

## Challenges and Trade-Offs

**Technical.** FHE is slow relative to plaintext EVM. V6.5 keeps on-chain FHE state lean (one `euint64` per entry). RIP-7212 is assumed present; pre-Pectra chains are out of scope. Two ACL patterns — a self-call in the ledger to rebind `msg.sender` for FHE input verification, and transient-ACL forwarding at the vault — were needed to keep the stack un-whitelisted with Fhenix (FHEIP-0004, FHEIP-0009 canonicalize them).

**Economic.** 1% at cash-in only; internal transfers and cashouts are free. Paymaster is break-even on reasonable per-user volume. CCTP carries Circle's own fee (~1 bp mainnet L2, 0 on testnet).

**Regulatory.** Anonymity at the protocol layer is an open question. Response is selective disclosure via `FHE.allow` — the same grant the user gives themselves can be extended to an auditor. Stealth addresses are a standardized construction (ERC-5564). CCTP is permissionless Circle infrastructure. Z0tz inherits those postures.

**UX.** The user creates a passkey (biometric), receives a meta-address, interacts normally. Everything else — stealth derivation, HKDF, P-256 signing, paymaster, CCTP — is automatic. Lost passkey without recovery paths = lost balance. Same property every non-custodial wallet has.

## Relation to other approaches

The Fhenix *Fluton × Fhenix* announcement and the UTXO confidential-tokens proposal share Z0tz's core thesis — that meaningful privacy needs both confidentiality and anonymity, composed. They diverge on which primitive carries the anonymity load.

- **Fluton** binds encrypted EOAs inside per-user smart accounts and routes execution through an intent-solver network. A different indirection for the same goal of removing the user's persistent identity from the on-chain event stream.
- **UTXO confidential tokens** commit to amount-private transfers with public parties by design, on the thesis that composability with arbitrary smart contracts matters more than hiding the sender at the base layer. An optional zk-wormhole adds recipient anonymity. A deliberate trade in the opposite direction from Z0tz.
- **Z0tz V6.5** combines FHE, stealth addresses as protocol proxies, a sweeper that uniformizes shield attribution across the whole user population, a pooled vault, a pseudonymous passkey-derived ledger, automatic per-spend rotation, a paymaster, and CCTP-via-stealth. Running on three testnets.

Each system validates parts of what the others claim. Z0tz is a case study in what the composition-based branch looks like when it ships.

## What's distinctive about this case study

**No mixer pool.** Mixing is a side effect of every user's funds passing through the same sweeper contract. No deposit queue, no waiting period, no separate protocol.

**Six privacy mechanisms stacked.** FHE (confidentiality) + stealth (per-tx anonymity) + sweeper (uniform attribution at the shield step) + pooled vault (holder anonymity) + pseudonymous ledger (mapping-key anonymity) + automatic rotation (cross-tx unlinkability).

**Stealth-as-proxy generalizes.** CCTP is the first proof; the same template applies to any permissionless EVM protocol without modifying the protocol.

**Nine new contracts across three chains, one env var to roll back.** V6 contracts unchanged. Paymaster flag flip reverts the whole stack.

## Contribution to Fhenix and Ethereum

The concrete deliverables — nine contracts, three testnets, measured gas — are less important than the template that emerges from them. What Z0tz V6.5 contributes back to the ecosystem, independent of whether Z0tz itself scales:

1. **Stealth-as-proxy is a reusable composition primitive** for adding unlinkability to any permissionless protocol without modifying the protocol. CCTP is the first published instance; the same three parts (pre-stage → interact at stealth → post-mix through sweeper) apply anywhere.
2. **Sweeper-as-mixer is a reusable attribution primitive** that collapses per-user shield events into a uniform stream without a deposit pool or waiting period.
3. **Passkey-pseudonymous ledger keys remove the "persistent holder in a confidential-token contract" leak** that otherwise persists even under FHE. FHEIP-0001 generalizes the viewer-permit path; FHEIP-0003 canonicalizes the events so cross-dapp indexers work.

Fhenix supplies FHE. Circle supplies cross-chain transport. ERC-4337 supplies accounts. RIP-7212 supplies P-256 verification. Z0tz threads them so the user stays private through all of it. Every new piece of permissionless infrastructure becomes a composition target for the same template.

## Open research

Shared with Fluton and the UTXO proposal: network-layer privacy (TOR/NYM), confidential DeFi composition (protocol-side FHE adoption), pool-wide anonymity sets (compose with a ZK pool at a stealth transition), standard event schemas / permits / wire formats (covered by the FHEIP drafts in this repo).

## Closing

Privacy on public chains is not one technology. It is a stack — and the best contribution a single project can make is to show that the stack composes. Z0tz V6.5 is one wallet's attempt at that composition, sitting alongside Fluton's and the UTXO proposal's attempts, each validating part of what the others claim.

Confidentiality and anonymity, composed. Running on testnet today.
