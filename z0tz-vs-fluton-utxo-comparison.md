# Confidential-ledger design space: Z0tz V6.5, Fluton, and UTXO confidential tokens

A research comparison across three approaches to on-chain financial privacy. Same problem surface, different primitives, different trade-offs. The goal is to map the space — not to rank.

## 1. The privacy surface

At the application layer, blockchain privacy decomposes into four independent axes:

- **Amount confidentiality** — values are hidden.
- **Sender anonymity** — the originating identity is unlinkable.
- **Recipient anonymity** — the destination identity is unlinkable.
- **Cross-transaction unlinkability** — operations by the same user don't cluster into a fingerprint.

Plus two adjacent concerns no deployed protocol fully addresses:

- **Network-layer privacy** — IP addresses, RPC requests, mempool observation.
- **Composability** — whether the privacy layer breaks when the user interacts with protocols not designed with it in mind.

## 2. Fluton

Fluton is the architecture described in the Fhenix blog post *"Fluton × Fhenix: Confidentiality Meets Anonymity."* Three pieces:

- **FHE** for amount and parameter confidentiality (Fhenix CoFHE, with TEE-intermediated decryption at cross-provider transitions).
- **Smart accounts** for anonymity: a fresh smart-account wallet per user, with the original EOA stored inside the contract as an encrypted ciphertext.
- **Anoma solvers + Fluton relayers** for routing: small transfers through relayers, large multi-step operations through the intent-solver network.

An **adapter** batches encrypted inputs across users and submits bundled transactions downstream, blurring individual metadata inside the anonymity set.

Fluton is a published architecture. The announcement is explicit about this: FHE's performance overhead (~1000× native), regulatory uncertainty, and a user-education curve are acknowledged trade-offs. No deployed contracts, testnet results, or gas measurements are referenced in the public materials.

## 3. UTXO confidential tokens

The UTXO confidential-tokens proposal (HackMD, @georgeh) takes the opposite route: amounts private, parties public. Three components:

- **Base confidential contract** — commitment Merkle tree, nullifiers, ZK proof verification.
- **Token wrapper** — deposits, withdrawals, plaintext ↔ confidential conversion.
- **ZK circuit** — proves valid UTXO transactions: Merkle membership, amount conservation, no double-spend.

The privacy model is **selective disclosure** — "confidential transfers expose the sender and recipient addresses onchain while keeping the transfer amounts private." The benefit of this trade: the circuit doesn't verify signatures, so any contract (multisig, vault, policy-gated account) can execute confidential transfers. The anonymity set extends to all addresses, not just EOAs.

An optional **zk-wormhole** hides the actual recipient behind a decoy address, restoring recipient anonymity at a per-transfer cost.

Scope is token transfers. Cross-chain, gas metadata, and key management are out of scope by design.

## 4. Z0tz V6.5

Z0tz V6.5 ships a wallet-level privacy stack on Base / Eth / Arb Sepolia. Six primitives compose:

- **FHE (Fhenix CoFHE)** — `euint64` balances in `Z0tzPrivateLedger`; every internal transfer stays encrypted end-to-end.
- **Stealth addresses (ERC-5564 / 6538)** — one-time addresses at cash-in and cash-out; the sender never sees the recipient's wallet, the target never sees the source ledger.
- **Sweeper as a mixing layer.** `Z0tzPrivateSweeperV2` is the only `msg.sender` for every shielding operation across the entire user population. An observer watching shield events sees a uniform stream of `sweeper → vault` operations with the sweeper as caller and amounts encrypted into the ledger on receipt. No mixing pool, no exit queue — the mixing is a side effect of every user's funds passing through one contract that looks identical for everyone.
- **Pooled vault + pseudonymous ledger** — `Z0tzPrivateLedgerVault` is the only FHERC-20 holder for the whole population. Per-user balances live in `Z0tzPrivateLedger.entries[ledgerId]` under a `bytes32 ledgerId = HKDF(passkey, "z0tz-ledger-id", vault, nonce)`. The ledgerId has no on-chain link to any Ethereum address and rotates automatically on every spend.
- **Paymaster** — 1% token fee at cash-in, zero ETH held by the user.
- **CCTP V2 via stealth pair** — cross-chain transport runs on Circle's permissionless infrastructure. Z0tz wraps it in a stealth pair so the bridge's public events name stealths, not the user.

Auth is **P-256 passkey over RIP-7212** (~3.5K gas per verify) direct at the ledger. No ERC-1271, no smart-account call path for confidential-token operations.

### The CCTP integration is a composability proof

Cross-chain is the hardest case for a composition-based privacy layer — two chains running simultaneously, a mandatory plaintext window because FHE ciphertext handles are chain-scoped, no privacy-friendly bridge alternative in production. If the pattern holds there, it holds for simpler cases.

What V6.5 proves: a stealth address can serve as the user's *proxy* when interacting with any permissionless protocol. The stealth, not the user's account, is what the external protocol sees. The sweeper then post-mixes the protocol's output back into the user's ledger under a pseudonymous id. The external protocol doesn't change; the user's persistent identity stays out of its event stream. Three parts:

1. **Pre-stage at a stealth** — route funds to a one-time stealth on the same chain, amount encrypted.
2. **Protocol interaction at the stealth** — the stealth calls into the external protocol. Its events name the stealth.
3. **Post-mix through the sweeper** — the output flows back through the shared sweeper contract, attributed to every other user's activity.

CCTP is the first working instance. The same three parts apply without modification to any permissionless EVM protocol — a DEX swap, a lending deposit, an NFT mint, a governance vote, an airdrop claim. That generality is what makes this a contribution to the Fhenix-and-Ethereum privacy stack rather than a single-wallet feature.

### Tezcatli vault composition is the second working instance

The second concrete instance of the stealth-as-proxy template is Tezcatli's confidential vault stack — an FHE-encrypted vault primitive (share/asset accounting on `euint64` handles) with an Aave V3 strategy adapter, plugged into V6.5 with no wallet-side privacy changes. A fresh DeFi stealth derives from `(passkey, originChainId, vaultChainId, vaultAddress, index)`, the ledger debits to it, the stealth deposits, and the stealth dies. The vault sees one ephemeral depositor per deposit and never learns the user's smart-account address. On withdraw the wallet routes funds back to the chain the position was opened from (ledger A → vault on B → CCTP → ledger A), reusing the same cross-chain template CCTP cashouts already use.

This matters as comparison evidence because a vault has internal state CCTP doesn't — shares, principal, fees, snapshots — and the wallet has to display all of those without leaking through a local cache or a per-position state map. V6.5's DeFi page reads four numbers per position straight from chain on every render (`principalDepositedOf`, `netPositionSnapshotOf`, `pendingYieldSnapshotOf`, plus the live Aave APY). No local cost-basis cache, no SQLite of historical events, no client-side bookkeeping that could leak through a logfile.

### Compliance at the integration boundary, not the wallet

The Tezcatli composition also lands the part of the privacy stack the V6.5 architecture deliberately did not address: an explicit compliance posture. The position is that a wallet that hides amounts and identities cannot also be the place that decides whether a sanctioned address gets to use the system; that decision belongs at the boundary. Three layers:

- **`Z0tzComplianceGate` (FHEIP-0010, on-chain).** Pure predicate consulted at every shield and unshield. `canShield` / `canUnshield` answer yes/no with a typed reason code (0..7). The gate has zero token-moving authority. Composed of a KYC registry (yes/no oracle, no PII), a sanctions block-list, and an append-only depositor registry. Default-permissive (empty deny-list ⇒ everyone allowed); `enabled` defaults to false during bring-up. Two-step admin transfers (Ownable2Step style).
- **Geofencing (relayer HTTP layer, default-on).** Restricted regions hit a 403 before anything reaches chain. Country list mirrors the published OFAC sanctions set.
- **KYC supplier (off-chain, opt-in per integration).** Bridges to Sumsub, Persona, Chainalysis KYT when an SDK integrator needs it. Z0tz the wallet never demands KYC from end users.

The wallet pre-flights the gate via `eth_call` before paying any gas, so a denied operation costs nothing and the GUI surfaces a typed reason instead of a raw selector. Z0tz never holds, freezes, or auto-returns flagged funds; there is no admin who can release seized assets and no compliance custody vault. The gate's job is to refuse — when it does, nothing moves and the user keeps their keys.

## 5. Comparison matrix

Blank = out of declared scope.

| Axis | Fluton | UTXO confidential tokens | Z0tz V6.5 |
|---|---|---|---|
| Amount confidentiality | FHE (planned) | ZK commitments | **FHE (working, testnet)** |
| Sender anonymity | Smart-account rotation | Public by design | Stealth + pooled vault |
| Recipient anonymity | Smart-account rotation | Wormhole (optional) | Stealth + pseudonymous ledgerId |
| Cross-tx unlinkability | Fresh account per interaction | Per-note | Auto-rotation on every spend (~25K gas) |
| Gas metadata | Solver sponsorship (planned) | — | Paymaster (sub-cent, 1% fee) |
| Cross-chain | Research frontier | — | CCTP V2 + stealth pair (working, three flows) |
| Anonymity set | Adapter batching | All addresses | Pooled vault + shared sweeper |
| Composability with unmodified protocols | Adapter | UTXO semantics diverge from ERC-20 | Stealth pre-stage + sweeper post-mix |
| Confidential DeFi composition | Solver-routed (planned) | Out of scope | Tezcatli vault on Aave V3 (working, arb-sepolia) |
| Compliance posture | Not specified | Selective disclosure (per-tx) | On-chain gate (FHEIP-0010) + geofence + opt-in KYC |
| Key model | Not specified | Not specified | Passkey (P-256, WebAuthn-compatible) |
| Deployment | Architecture paper | Proposal | Testnet on 3 chains, gas measured |
| TEE dependency | Yes (cross-provider) | No | No |
| Permissionless bridge | — | — | Yes (Circle CCTP V2) |

Measured on V6.5 (Base Sepolia, 5 gwei, April 2026):

| Flow | Gas | L2 cost |
|---|---:|---:|
| Cash-in (stealth → sweeper → vault → ledger credit) | 591 K | $0.012 |
| Internal transfer + pseudonym rotation | 405 K | $0.008 |
| Same-chain cashout (ledger → vault → stealth) | 683 K | $0.014 |
| Cross-chain bridge via CCTP | 1.5 M | $0.030 |

## 6. Validation, not competition

The three approaches converge on the same thesis — that meaningful privacy on public chains requires both confidentiality and anonymity, composed — and diverge on which primitives carry the load. Each system validates a part of the shared thesis that the others reinforce.

- **Fluton** is right that anonymity needs to sit at the account layer, not just the transaction layer. Its encrypted-EOA binding inside a smart account is a different answer to the same question Z0tz answers with stealth + pseudonymous ledger. Both approaches pull the user's real identity out of the on-chain event stream; they just disagree on whether the indirection is encryption of the binding or one-time derivation of the key.

- **UTXO confidential tokens** are right that confidentiality and composability can coexist without hiding every party. Selective disclosure lets any contract — multisigs, policy vaults, DAOs — execute confidential transfers without the circuit needing signature verification. Z0tz uses a different trade (stealths hide parties, a pseudonymous ledger hides the mapping key, the wrapped FHERC-20 is untouched), but the underlying observation — *privacy is a set of dials, and turning them independently is better than one monolithic setting* — is shared.

- **Z0tz** contributes a working instance of a composition-based approach. The combination stealth-as-proxy (for any permissionless protocol) plus sweeper-as-mixer (to uniformize the output attribution) is an architectural template that external protocols don't need to know about. CCTP is the first test case; the same template applies anywhere the user needs unlinkability with an unmodified protocol.

A practical composition across the three: a Z0tz user who needs pool-wide anonymity at a specific step can cash out from the ledger into a stealth, deposit into a UTXO + zk-wormhole pool, and withdraw later. That's the same three-part template V6.5 uses for CCTP, applied to a ZK pool instead of a bridge. None of the three approaches has to change to make this work.

## 7. Open research across all three

- **Network-layer privacy.** None address RPC metadata or mempool observation at the deployed layer. Orthogonal but necessary.
- **Composable confidential DeFi.** FHE inside a DEX / lending protocol requires protocol-side adoption. A confidential-ERC-20 standard plus solvency-checked primitives (FHEIP-0002) would lower the threshold.
- **Standard event schemas.** Each system invents its own; no indexer renders balances cross-dapp without custom parsing. FHEIP-0003 unifies this.
- **Auth-scheme registries.** FHEIP-0005 / FHEIP-0006 let passkey, EOA, and ERC-1271 permits coexist. Any of the three systems benefits.
- **Gas schedules for FHE.** All three rely on FHE's performance profile; a published schedule lowers the barrier for new dapps.
- **Compliance via view-key permits.** `FHE.allow` to an auditor is the native selective-disclosure primitive. Shared ground for all three.

## 8. What Z0tz contributes back

Z0tz V6.5 is one wallet; it is not the privacy layer for Ethereum. What the V6.5 deployment contributes, beyond its own users, is empirical evidence that three specific ideas generalize:

1. **Stealth addresses work as proxies for arbitrary permissionless protocols.** The CCTP integration demonstrates the hardest case (two chains, plaintext bridge window). The same pattern applies to DEXes, lending markets, NFT mints, governance — any protocol where the user would normally have to reveal a persistent address.

2. **A single mixing contract (the sweeper) plus a pooled holder (the vault) collapses the observable on-chain graph of shielded flows into a stream indistinguishable across users.** No pool, no queue, no separate protocol. Just the fact that every user's funds pass through the same `msg.sender` at the shield step.

3. **Passkey-derived pseudonymous accounting keys remove the "persistent holder in a confidential-token contract" leak.** The ledger is keyed by `HKDF(passkey, ...)`, not by an Ethereum address. Re-derivable on any device, unlinkable on-chain.

Together these three give a reusable template — stealth-as-proxy, sweeper-as-mixer, passkey-pseudonymous-ledger — that adds privacy to the Ethereum ecosystem by composing with what already exists (Fhenix CoFHE, ERC-4337, RIP-7212, Circle CCTP, ERC-5564). That kind of additive, composition-based contribution is what we think Ethereum's privacy roadmap actually needs: not one monolithic privacy protocol that the rest of the stack has to accommodate, but a small set of composition primitives that let privacy travel with the user into any permissionless protocol they want to interact with.

## 9. Summary

Fluton binds encrypted EOAs inside smart accounts and routes through an intent-solver network. UTXO confidential tokens ship amount-confidential transfers with public parties by design and layer wormhole anonymity on top. Z0tz V6.5 combines FHE, stealth proxies, sweeper mixing, a pooled vault, a pseudonymous ledger, auto-rotation, and CCTP-via-stealth into one wallet running on three testnets today.

Different primitives, different coverage, shared thesis. The space grows by composition, and each system validates part of what the others claim.
