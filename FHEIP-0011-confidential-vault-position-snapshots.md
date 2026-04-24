---
fheip: 0011
title: Per-account encrypted position snapshots for confidential vaults
description: Standardize a small set of `view`-only encrypted-handle getters on confidential strategy vaults — `principalDepositedOf`, `grossPositionSnapshotOf`, `netPositionSnapshotOf`, `pendingYieldSnapshotOf`, `pendingFeeSnapshotOf`, `snapshotTimestampOf` — so wallets can render rich per-position dashboards (principal vs. earnings, lock countdown, fee preview) without per-vault bespoke decoders, and so cost-basis tracking can be sourced from chain truth instead of a leaky local store.
author: 0xOucan (@0xOucan)
discussions-to: TBD
status: Draft
type: Standards Track
category: ERC
created: 2026-04-23
requires: FHE.sol (cofhe-contracts), FHEIP-0001, FHEIP-0008
---

## Abstract

Confidential vaults today expose a single user-facing handle: `confidentialSharesOf(account)` (or equivalent) returning an `euint64` of share balance. That's enough for a balance display but useless for a real DeFi dashboard, which needs to render principal, accrued yield, pending fees, lock state, and "value at last sync" — information the vault already has internally as encrypted state.

This FHEIP standardizes six per-account `view` getters that return existing encrypted handles, so any wallet can build a "Aave-grade" position card on a confidential vault with no per-vault customization. The getters are `view`-only and do not mutate ACL state — wallets pair them with FHEIP-0001 viewer permits to decrypt off-chain. The proposal also fixes a real, currently-shipping mismatch: the Tezcatli Confidential Vault's source code on GitHub advertises these getters, but the deployed bytecode on arb-sepolia does not include them, so wallets that try to call them revert. Standardizing the surface makes the deployed-vs-source contract testable and gives wallet authors a stable target.

## Motivation

Z0tz's DeFi dashboard targets parity with Aave / Compound UX: each position row shows principal, current value, earnings to date, hypothetical APY-projected yield, lock countdown, and pending fees. To get those numbers from a confidential vault today, the wallet has to either:

1. **Track everything off-chain.** Record each deposit's underlying-amount in a local store, sum into a "cost basis", subtract from the decrypted current value. This works but is fragile: the local store can desync (vault redeployments reuse the same `vaultId`, devices that don't have the local store can't render history, dust truncation skews the sum). Z0tz hit phantom "drift -5 USDC" rows from exactly this kind of stale cost-basis store after a vault redeploy.
2. **Reconstruct from events.** Walk every `Transfer` of the underlying into the vault scoped by user. Possible but slow, requires per-vault event schema knowledge, and breaks when the vault uses non-standard event topics.
3. **Read raw vault internals.** Reverse-engineer the storage layout. Brittle and unsafe.

The right answer is for the vault to expose the encrypted handles it already maintains. They're encrypted; reading them through a viewer permit is the user's prerogative. The getters are `view`-only, so they don't widen the attack surface.

A second motivation is **cost-basis honesty**. A confidential strategy vault that updates positions across blocks (e.g., Aave V3 yield accrual) holds a "snapshot timestamp" alongside the encrypted gross/net values. Without surfacing that timestamp, a wallet rendering "earnings: 1.234 USDC" cannot tell whether the figure is fresh or two weeks stale. Pairing each snapshot with its timestamp lets wallets show "earnings: 1.234 USDC (synced 4h ago)" and prompt re-sync when stale.

### Use cases

- Wallet dashboards (Z0tz, future FHE-position aggregators, mobile portfolio trackers)
- Tax / accounting tools — principal vs. earnings split is fundamentally a tax question
- Lock countdown UI — "you can withdraw in 6d 4h" requires both current time and the per-account lock-end snapshot
- Multi-vault aggregate views — one decryption call per position, summed off-chain
- Cost-basis reconciliation across device migrations — the user can rebuild full DeFi history from `passkey + chain`, with no local store required

### Non-goals

- Defining the strategy semantics (what "principal" or "yield" means inside the vault). The vault decides; the getters surface its definitions.
- Mandating a specific snapshot frequency. Snapshots can be per-tx, per-block, per-epoch — the timestamp lets wallets adapt.
- Handling vaults that genuinely have no notion of yield (e.g., pure escrow). Such vaults MAY return zero handles or omit the views.
- Specifying the relationship to share price / virtual-share inflation defenses. The getters return the user-facing values; share-price math stays internal.

## Specification

### 1. Required encrypted-handle getters

```solidity
interface IConfidentialVaultPositionSnapshots {
    /// @notice Total principal `account` has deposited, in underlying decimals.
    /// @dev    Cumulative in. Withdraws DO NOT decrement principal directly;
    ///         instead, partial withdraws scale principal proportionally to
    ///         shares-burned/shares-held. Vaults MUST document their accounting.
    function principalDepositedOf(address account) external view returns (euint64);

    /// @notice Gross position value at last snapshot, in underlying decimals.
    ///         Excludes fees the user would owe on withdraw.
    function grossPositionSnapshotOf(address account) external view returns (euint64);

    /// @notice Net position value at last snapshot, in underlying decimals.
    ///         What the user would receive if they withdrew everything right now,
    ///         after all vault-side fees.
    ///         INVARIANT: netPosition ≤ grossPosition.
    function netPositionSnapshotOf(address account) external view returns (euint64);

    /// @notice Earnings accrued since the last principal increase, snapshotted.
    /// @dev    Definition is vault-specific. Common: gross - principal, clamped at 0.
    ///         Vaults MUST document the formula in NatSpec.
    function pendingYieldSnapshotOf(address account) external view returns (euint64);

    /// @notice Vault-side fee that would be deducted on a full withdraw right now.
    /// @dev    INVARIANT: pendingFee == grossPosition - netPosition.
    function pendingFeeSnapshotOf(address account) external view returns (euint64);

    /// @notice Block timestamp at which the snapshots above were last refreshed.
    /// @dev    Plaintext, NOT encrypted. Wallets compute "freshness" off-chain.
    ///         Vaults MUST update this whenever any of the encrypted snapshots change.
    function snapshotTimestampOf(address account) external view returns (uint64);
}
```

All five encrypted getters return live `euint64` ciphertext handles. The vault MUST grant the `account` permanent ACL access to each handle — i.e., `FHE.allow(handle, account)` must be in effect — so the user can decrypt via FHEIP-0001 viewer permits without an additional ACL setup tx.

Calling these views from an account that doesn't hold the position MUST NOT revert; it MUST return the empty / zero ciphertext handle for an account with no position. This lets wallets scan `vault.principalDepositedOf(stealth_i)` for `i ∈ [0, 8)` (per FHEIP-0001 stealth derivation) without needing an existence check first.

### 2. Lock-state getters (optional but recommended)

```solidity
interface IConfidentialVaultLockState {
    /// @notice Earliest unix timestamp at which `account` can fully withdraw.
    ///         Returns 0 if no lock applies.
    function withdrawUnlockTimeOf(address account) external view returns (uint64);

    /// @notice Vault-wide minimum delay between deposit and withdraw, seconds.
    ///         Plaintext. Wallets render "min lock: 7 days" without per-account decryption.
    function minWithdrawDelay() external view returns (uint64);
}
```

Lock state is plaintext because (a) lock periods are typically a vault parameter, not a privacy-sensitive per-user value, and (b) wallets need to render the countdown without a decryption round-trip on every refresh. Vaults whose lock model genuinely is per-user-private MAY return `type(uint64).max` to signal "private; query via viewer permit on a separate handle".

### 3. ACL grant timing

The vault MUST call `FHE.allow(handle, account)` for each of the five encrypted handles in §1 at the moment the position is first opened (first `shield` / first `confidentialTransferAndCall` from `account`). The grant is permanent until governance-side `revokeAllow`. The vault MUST NOT re-grant on every snapshot update, only on first open — re-granting is gas-wasteful and indistinguishable from the initial grant for wallet purposes.

If a vault does NOT grant ACL on the handles, the views still work (they return handles), but `decryptForView` will fail at the threshold network. Vaults that intentionally withhold ACL (e.g., for restricted custody products) MUST document this in NatSpec.

### 4. Wallet rendering pattern

```typescript
// 1. Read snapshots in parallel.
const [
    encPrincipal, encGross, encNet, encYield, encFee, ts, unlockAt
] = await Promise.all([
    vault.principalDepositedOf(stealth),
    vault.grossPositionSnapshotOf(stealth),
    vault.netPositionSnapshotOf(stealth),
    vault.pendingYieldSnapshotOf(stealth),
    vault.pendingFeeSnapshotOf(stealth),
    vault.snapshotTimestampOf(stealth),
    vault.withdrawUnlockTimeOf(stealth),
]);

// 2. Decrypt via one viewer permit covering all five handles.
const permit = await issuePermit({
    ctHashes: [encPrincipal, encGross, encNet, encYield, encFee],
    viewer:   passkeyViewer,             // FHEIP-0001
    deadline: now + 600,
});
const [principal, gross, net, yield_, fee] = await decryptBatch(permit);

// 3. Render.
render({
    principal, gross, net,
    earnings:        gross - principal,    // honest definition; matches §1
    fee,
    snapshotAge:     now - ts,
    canWithdrawIn:   max(0, unlockAt - now),
});
```

A wallet MUST NOT render mixed plaintext / "deduced" values without a sanity check: if `pendingFeeSnapshotOf - (grossPositionSnapshotOf - netPositionSnapshotOf)` is non-zero (after decryption), the vault has violated the §1 invariant and the wallet SHOULD warn the user.

### 5. Aggregate scanning

A wallet MAY discover all of a user's positions by walking the FHEIP-0001 stealth derivation indices. For each `i ∈ [0, MAX_INDEX)`:

```solidity
address stealth_i = deriveStealth(passkey, chainId, vaultAddress, i);
euint64 principalHandle = vault.principalDepositedOf(stealth_i);
if (FHE.isZero(principalHandle)) continue;       // no position
// else: this index has a position; render it.
```

Vaults SHOULD cap `MAX_INDEX` at 64 in wallet conventions. Z0tz uses 8 today and will lift it as needed.

## Rationale

**Why expose all five values when wallets could derive some from others?** The values are not algebraically reducible: `yield = gross - principal` is one common definition, but a vault that compounds at the strategy level (e.g., Aave aTokens) may define yield as `gross - principal_at_last_compound` rather than `gross - cumulative_principal`. Letting the vault publish its own definition removes wallet-side guessing. The `gross - net = fee` invariant IS algebraic — that's why `pendingFee` is included as a "convenience" view derivable by the wallet, but having it explicit lets the vault enforce the invariant in storage.

**Why `principalDepositedOf` instead of letting the wallet sum its own deposits?** Two reasons. (1) Devices that don't have the local store (new device, restored passkey) need to rebuild without scanning years of events. (2) Vaults often apply "principal scaling" on partial withdraws (proportional reduction); reproducing that off-chain duplicates vault accounting. Letting the vault be the source of truth avoids drift.

**Why plaintext snapshot timestamp?** Encrypting the timestamp would force a decryption round-trip just to render "synced 4h ago", which is silly. Timestamps are not privacy-sensitive in any threat model we surveyed — they leak that "this account had activity at time T", which the underlying chain already exposes via tx history.

**Why `uint64` for handles and timestamps?** Matches FHEIP-0008's `wrapperDecimals ≤ 8` cap and Solidity's standard timestamp width. Future widening to `euint128` would require a separate FHEIP (and a new FHE primitive — out of scope).

**Why not include APY?** APY is a strategy property, not a position property. Wallets read it once per vault from the underlying strategy (Aave's `currentLiquidityRate`, Compound's `supplyRatePerBlock`, etc.). Mixing it into the per-position interface would force every vault to publish a synthetic APY, which then disagrees with the strategy's authoritative figure.

**Why required ACL grant on first open?** The alternative — granting on every snapshot update — wastes gas and pollutes the ACL with redundant entries. First-open-grant is the minimal correct posture: the user gets exactly the access they need, exactly once.

**Why offer `withdrawUnlockTimeOf` separately from §1?** Lock state is plaintext, snapshots are encrypted. Mixing the two getter shapes invites confusion. Splitting also lets vaults that have snapshots but no lock (or lock but no snapshots) implement just the half they support.

**Why surface the github-vs-bytecode mismatch in this FHEIP?** Because the standardization itself fixes it: a vault that conforms to this FHEIP MUST deploy the views, not just declare them. Wallets can then test conformance with a single eth_call per view at deploy time and refuse to integrate non-conforming deployments. Z0tz's DeFi tab includes exactly this "Tier 1 / Tier 2" detection — Tier 1 (this FHEIP's full surface) lights up only when the views actually return data; Tier 2 (legacy `confidentialSharesOf` + local cost basis) is the fallback.

## Backwards Compatibility

Additive. Vaults exposing only `confidentialSharesOf` continue to work; wallets fall back to the local-cost-basis path with reduced fidelity. As vaults adopt this FHEIP, the same wallet code lights up the rich path automatically — Z0tz's `defi-decrypt.ts` already uses `Promise.allSettled` so missing views degrade gracefully.

The Tezcatli vault on arb-sepolia: source code at the listed GitHub repo declares all five views, but the deployed bytecode (as of 2026-04-23) does not include them; calls revert. A redeployment that matches the source-declared surface would conform to this FHEIP. This FHEIP's existence does not retroactively fix the deployed vault; it gives the redeploy a target to hit and gives wallets a posture to detect and gracefully degrade.

## Security Considerations

1. **ACL leak via stale grants.** A vault that grants ACL at first open MUST NOT subsequently widen the grant (e.g., to a different account). If the user's stealth EOA ownership transfers (not a Z0tz pattern, but possible in other wallets), the old owner retains `FHE.allow` access. Vaults MAY expose `revokePositionAccess(account)` for governance-driven cleanup; this FHEIP does not require it.
2. **Snapshot freshness.** A wallet rendering "earnings: X" from a snapshot that's two weeks old is showing a misleading number. Wallets MUST display the snapshot age and SHOULD warn when age exceeds a threshold (Z0tz uses 24h). Vaults SHOULD update snapshots at minimum every deposit / withdraw and SHOULD support an unauthenticated `refreshSnapshot(address)` that any caller can poke (the underlying strategy's read-only state typically suffices to compute fresh values, so no privilege is required).
3. **Empty-handle decryption.** Calling `decryptForView` on a zero handle produces a zero plaintext, not a revert. Wallets MUST treat "principal == 0 and gross == 0" as "no position", not "position is zero" (which would be unreachable in practice but indistinguishable on the wire).
4. **Handle reuse across positions.** A vault MUST allocate fresh handles per `(account, position)`. Reusing a handle across two accounts (e.g., a vault-shared "total fee pool" handle) and then granting both accounts ACL on it would let either decrypt the other's data through a permit on the same handle. This FHEIP's getters are per-account by design; vault implementers MUST NOT shortcut this.
5. **Invariant violation as a privacy oracle.** If a vault's `pendingFee + net != gross` after decryption, the vault is buggy or malicious. A wallet that reports the discrepancy off-chain (telemetry, error logs) leaks decrypted values. Wallets SHOULD surface invariant violations to the user but MUST NOT exfiltrate the decrypted figures.
6. **MAX_INDEX timing channel.** Walking 0..63 indices in parallel reveals to an RPC operator that "this caller has a Z0tz wallet" via the request pattern. Wallets SHOULD interleave the scan with other reads or use a private RPC. This is a generic privacy concern, not specific to this FHEIP, but worth noting because the position scan has a distinctive shape.
7. **Plaintext lock-time correlation.** `withdrawUnlockTimeOf(stealth_i)` reveals when a position was opened (deposit_time + minWithdrawDelay). Two queries from the same RPC client can correlate two stealths to the same passkey if they both have lock times within seconds of each other. Wallets that need extra unlinkability MAY randomize cross-stealth scan timing; the threat is generally below the bar that motivates a FHEIP-level mitigation.
8. **Vault upgradeability.** A vault that upgrades and changes the meaning of a snapshot field (e.g., redefines `pendingYieldSnapshotOf` from "since-last-deposit" to "since-inception") silently breaks every wallet's rendering. Vaults SHOULD treat the §1 surface as part of their stable ABI and bump a major version on incompatible changes.

## Reference Implementation

- Source declaration (target): the Tezcatli Migrator vault contract on GitHub declares all five getters in the `IVaultViews` mixin pattern. See `Z0tz/cli/src/core/defi-decrypt.ts` for the wallet-side decoder that targets this surface.
- Wallet-side graceful fallback: [`Z0tz/cli/src/core/defi-decrypt.ts`](https://github.com/0xOucan/Z0tz/blob/DeFi/cli/src/core/defi-decrypt.ts) — `Promise.allSettled` over all five getters, synthesizes a `VaultPositionDecrypted` record with optional Tier-1 fields populated only when the calls succeed.
- Two-tier dashboard rendering: [`Z0tz/gui/src/renderer/pages/DeFi.tsx`](https://github.com/0xOucan/Z0tz/blob/DeFi/gui/src/renderer/pages/DeFi.tsx) — Tier 1 (this FHEIP's full surface) shows principal / gross / net / yield / fee / freshness; Tier 2 falls back to local cost-basis with a clear "low fidelity" badge.
- Stealth-index scanner: [`Z0tz/cli/src/core/defi-scan.ts`](https://github.com/0xOucan/Z0tz/blob/DeFi/cli/src/core/defi-scan.ts) — walks indices 0–7 of the FHEIP-0001 derivation per vault per chain, applying the §5 pattern.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
