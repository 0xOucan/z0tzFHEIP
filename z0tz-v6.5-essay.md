# Privacy is a stack, not a feature

*An essay on how Z0tz V6.5 composes FHE, stealth addresses, a pseudonymous ledger, and a permissionless bridge into a single wallet where the user's identity is absent from every confidential-token event.*

## The persistent belief

There is a persistent belief, in the discourse around blockchain privacy, that one technology solves the problem. Encrypt the balances. Hide the amounts. Ship it.

But privacy is not a feature you bolt onto a wallet. It is an architecture — a series of choices at every layer, each one closing a leak the others leave open. Miss one, and the rest unravel.

Fully Homomorphic Encryption is a breakthrough. Balances stay hidden. Transfer amounts become ciphertext. Smart contracts compute on data they never see. But FHE encrypts *computation* — it does not encrypt the *user*. Addresses are public. Gas payments leak timing. Shielding and unshielding are transparent checkpoints. Cross-chain activity creates linkable traces that undo everything the encryption achieved.

Z0tz is built on the premise that all of these leaks matter, and closing them requires solving each one, then composing the solutions.

## The leak V6 still left behind

V6 was a complete privacy stack by every reasonable measure. Encryption at the FHE boundary. Stealth addresses at the edges. A sweeper to mix the wrap step. A paymaster so users never touch ETH. Account abstraction so there is no seed phrase. A threshold network for decryption without hardware trust. Circle's CCTP V2 wrapped in stealth pairs so the bridge boundary belonged to someone else's permissionless infrastructure.

But one leak remained. Inside the FHERC-20 wrapper, the user's balance lived in a mapping keyed by their smart-account address. The amount was encrypted; the fact that *this address was a holder* was not. Anyone watching the contract could see which accounts were active, when they moved, and — correlating inbound stealth activity with outbound stealth activity through the user's account — reconstruct parts of the graph the stealth layer was built to hide. For a wallet that claims unlinkable identity, that graph is the last persistent identity surface.

V6.5 closes it.

## The architectural move

Stop having the user's smart account be the FHERC-20 holder. Hold all the FHERC-20 tokens in a pooled vault, and track per-user balances in a separate accounting contract whose mapping key is not an Ethereum address.

The pattern is old — every treasury system since the first vault contract splits the holder from the ledger. The holder is small, audit-friendly, and has no logic beyond "accept tokens from the authorized inbound, send to the authorized outbound." The ledger has the per-user accounting, the access control, the upgrade flexibility. Nothing in the ledger touches the actual token; it just authorizes the holder to move it.

In V6.5 the holder is `Z0tzPrivateLedgerVault`. The ledger is `Z0tzPrivateLedger`. The vault is the only address that appears as a holder in any FHERC-20 contract, no matter how many users the system has. The ledger holds per-user encrypted balances under a `bytes32 ledgerId` derived via HKDF from the user's passkey — not from any Ethereum address, re-derivable on any device that has the passkey, unlinkable on-chain.

The third piece is the sweeper. `Z0tzPrivateSweeperV2` is the only `msg.sender` for every shield operation across the entire user population. When anyone deposits, the events that hit the FHERC-20 and vault contracts have the sweeper as the caller and the vault as the target — uniform shape, same addresses, every user. That uniformity is the mixing: there is no pool to deposit into, no queue to wait on, no separate protocol to trust. The mixing is just a side effect of routing every user's funds through one contract that looks identical for everyone. It costs nothing above a normal tx; it was a tx that had to happen anyway.

The smart account still exists. It holds recovery state, signs CCTP interactions, works the way V6 specified. It just does not appear in any confidential-token event. That interaction — credit, debit, transfer, cashout — happens at the ledger, signed by the passkey directly via RIP-7212, with the relayer as `msg.sender`.

## What the user does

Almost nothing different. The wallet still accepts payments to a stealth address, still shows an encrypted balance the passkey can decrypt, still sends to other Z0tz users (now identified by a meta-address encoding a `ledgerId`), still cashes out to plaintext USDC at a normal address.

What changes under the hood: the wallet derives `ledgerId = HKDF(passkey, "z0tz-ledger-id", vault, nonce)` to find the balance. Spends sign a `SpendOp` with the passkey. Cash-ins credit the recipient's `ledgerId` via the sweeper. The smart account never appears.

On every spend the wallet quietly rotates the `ledgerId` to the next HKDF nonce. The contract performs the rotation atomically — debit old, credit change to new, delete old — for about twenty-five thousand extra gas. The user never sees it. The id they had at the start of the day is not the id they have at the end.

## The ledger is where the work lives

Each entry is three fields: an FHE-encrypted `euint64` balance, a `bytes32 pubkeyHash = keccak(P-256 pubkey)`, and a monotonic `uint256 nonce`. Authentication is direct P-256 RIP-7212 verification at the ledger — no ERC-1271, no smart-account indirection. RIP-7212 is live on all three target chains post-Pectra at ~3.5K gas per verify.

The op digest binds everything that matters: a typehash, chainid and ledger address, old and new ledger ids, destination ids and viewers, destination address for cashouts, the encrypted amount's ciphertext handle (so the relayer cannot substitute a different ciphertext), nonce, deadline. Twelve fields, all bound. The original draft omitted the ciphertext handle and was caught by the audit — fixed.

A spend verifies the deadline, checks the nonce and pubkey hash, verifies the P-256 signature, loads the encrypted amount through a self-call that flips `msg.sender` to the ledger itself so the CoFHE ACL binds correctly, computes `transferred = FHE.select(amount <= balance, amount, FHE.asEuint64(0))` — the privacy-preserving solvency pattern that silently falls back to zero — subtracts, bumps the nonce, atomically rotates if requested, and dispatches on the action. Internal transfers FHE-add to the destination; cashouts call `vault.confidentialTransferOut`.

## The flows

**Cash-in.** External sender → stealth → sweeper → vault → ledger entry under a pseudonymous id. The stealth signature commits to every routing choice. The sweeper pulls USDC, takes 1% for the treasury (try/catch, so a USDC blacklist never bricks the sweep), hands the rest to the vault for shielding, and credits the ledger with the actually-shielded amount. About 591K gas; the savings over V6 come from removing the smart-account UserOp leg.

**Cashout.** Two mixing layers, neither involving the user's wallet. The vault is the only FHERC-20 holder, so the event names `vault → stealth`. The stealth is one-time, so the target sees `stealth → target`. An observer watching the ledger sees `ledgerId debited X via cashout to stealth`. There is no on-chain path from `target` back to a smart account. The only correlation is `ledgerId → stealth → target`, and the ledgerId has no Ethereum link without the passkey.

**Cross-chain.** FHE ciphertext handles are chain-specific — they can't move. The bridge must transit through plaintext. The question is not whether plaintext exists; it's whether that plaintext can be linked to the user. Z0tz does not operate the bridge. Circle does, via CCTP V2. The twelve-step private bridge generates two stealths — one per chain — and routes the burn at the source stealth, the mint at the destination stealth, the sweep into the destination ledger. CCTP's public events connect two random addresses; neither is the user.

## Auto-rotation: free unlinkability

The remaining leak after V6.5 is intra-ledger pseudonymity. Same `ledgerId` over time becomes a cluster — a Twitter handle without a face.

Auto-rotation makes the cluster lifetime one op. On every spend the wallet derives the next id from `HKDF(passkey, vault, nonce+1)` and includes it as `newId`. The contract rotates atomically for about twenty-five thousand extra gas. For users who don't spend often, an idle epoch-based rotation runs monthly in the background.

What rotation can't do alone is hide the link `oldId → newId` in the rotation tx itself; the calldata names both. A future `rotateBatch` would bundle many users' rotations into one tx and force an N-permutation puzzle. Practical once there's enough traffic to amortize.

## Two ACL patterns that made it work

Two small Solidity patterns turned out to be the thing that made V6.5 work without special whitelisting from Fhenix.

The first is a self-call. CoFHE's ACL ties an encrypted input to whatever address was `msg.sender` at the moment `verifyInput` ran. Inside a function called by a relayer EOA, that's the relayer, not the ledger. The fix: `ledger.verifyAmount(InEuint64) external`, guarded by `msg.sender == address(this)`, invoked via `this.verifyAmount(op.amount)`. An external call — even to self — shifts `msg.sender`. Zero privacy cost, ~700 gas overhead. FHEIP-0009 canonicalizes this.

The second is transient-ACL forwarding at the vault. After the ledger grants the vault transient access to the encrypted amount, the vault grants the wrapped token the same transient access before calling it. One line: `FHE.allowTransient(encAmount, address(wrappedToken))`. The wrapped token never sees the user. FHEIP-0004 standardizes it.

Both patterns keep every contract's ACL story local and explicit at each boundary. No whitelist, no deferred-trust operator. The chain of addresses the user touches stays short: stealth for entry, pseudonymous ledger entry for storage, stealth for exit, with the vault pooling everyone in between.

## What V6.5 does not claim

Not a wormhole. Pseudonymity unlinkable to Ethereum identity is strictly better than "smart account is a known holder," strictly weaker than "one of ten thousand anonymous depositors." Users who need pool-wide anonymity at every spend should compose: cash out via the ledger into a ZK pool, withdraw later. Same three-part pattern V6.5 uses for CCTP.

Not a confidential-DeFi standard. Token privacy end-to-end, but FHE inside a DEX or lending protocol still requires the protocol to ship FHE primitives. What V6.5 changes is that the address calling the protocol can be a stealth that pulls funds from the vault for one operation and dies.

Not network-layer privacy. RPC still leaks IP. TOR/NYM is in the CLI, not the GUI. Encrypted RPC is a separate research item.

Not key recovery. Lost passkey is lost balance unless the user runs recovery through the `RecoveryModule` on their smart account, which can sign a `migrateBalance(oldId, newId)` op.

## The composition bet

The deeper claim behind V6.5 is not that the seven layers happen to exist. It is that once you have a stealth address (to give the external protocol a one-time identity to talk to) and a sweeper (to collapse every user's activity into a uniform stream when the funds come back), you have a general strategy for adding unlinkability to the rest of the on-chain world without modifying the rest of the on-chain world.

Pre-stage at a stealth. The external protocol sees the stealth, not the user. Interact with the protocol at the stealth. Whatever the protocol logs, it logs about the stealth. Post-mix through the sweeper. The result flows back through a contract that looks identical for every user, so the return leg is indistinguishable from everyone else's. The protocol does not need to know Z0tz exists. It does not need to implement any privacy feature. It does not even need to be a bridge — the same three parts wrap a DEX swap, a lending deposit, an NFT mint, a governance vote, an airdrop claim.

CCTP is the first concrete proof, and the hardest case: two chains, an unavoidable plaintext window, no privacy-friendly alternative in production. The source stealth burns through Circle's `TokenMessengerV2.depositForBurn`, the destination stealth receives through `MessageTransmitterV2.receiveMessage`, and the destination sweeper folds the mint back into the user's ledger under a pseudonymous id. CCTP's public events connect two random addresses; neither is the user. If the pattern works there, it works almost anywhere.

V6 closed the bridge layer by composing over someone else's permissionless infrastructure. V6.5 closes the holder layer by separating the token pool from the per-user accounting and replacing the mapping key with a passkey-derived pseudonym. What's left is the network layer and the in-protocol layer — both bigger projects than V6.5, both pointed at by the same composition pattern as the next step.

## What V6.5 validates

The Fhenix *Fluton × Fhenix* announcement and the UTXO confidential-tokens proposal share this essay's thesis — that meaningful privacy on public chains needs confidentiality and anonymity, composed — and answer it with different primitives. Fluton binds encrypted EOAs inside per-user smart accounts and routes through an intent-solver network. The UTXO proposal ships amount-private transfers with publicly-visible parties and layers an optional zk-wormhole on top for recipient anonymity.

V6.5 is not a refutation of either; it is a case study for the branch that picks stealth-addresses-as-proxies, a sweeper to mix the shield step, a pooled vault to remove the per-user holder, and a pseudonymous ledger to remove the mapping key. Each approach validates part of what the others claim: Fluton validates that the anonymity load has to sit at the account layer, not just the transaction layer; the UTXO proposal validates that confidentiality and contract-level composability can coexist without hiding every party; V6.5 validates that a composition-based approach over existing permissionless primitives can ship and run on three testnets with measurable gas.

## The contribution

The bet underneath all of this: you can build a privacy layer that is just a composition strategy, and that strategy keeps working as the ecosystem it composes over grows. Fhenix encrypts computation. Circle moves USDC. ERC-4337 abstracts accounts. RIP-7212 verifies P-256. ERC-5564 derives stealth addresses. Z0tz threads them together so the user stays private through all of it.

Three specific pieces of that thread generalize beyond Z0tz. Stealth-as-proxy gives any permissionless protocol a one-time identity to interact with. Sweeper-as-mixer uniformizes the attribution when the funds come back. Passkey-pseudonymous ledger keys remove the holder leak from confidential-token contracts. These are small primitives that let privacy travel with the user into any permissionless protocol they choose, without asking that protocol to know about privacy at all.

That is, we think, what Ethereum's privacy roadmap actually needs — not one monolithic privacy protocol that the rest of the stack has to accommodate, but a small set of composition primitives that add privacy to the ecosystem by routing through what already exists. Z0tz is one wallet's case study for that approach. Fhenix supplies the confidentiality. The rest is plumbing.

The stack is resilient precisely because it does not depend on any one project owning every piece. The user, at the end, still just holds a passkey. They click a biometric. The rest is cryptography composing with itself.
