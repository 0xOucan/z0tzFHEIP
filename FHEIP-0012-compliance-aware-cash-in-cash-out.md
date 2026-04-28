---
fheip: 0012
title: Compliance-aware confidential cash-in and cash-out boundaries
description: Standardize how an FHE-encrypted privacy stack consults a compliance gate (FHEIP-0010) at the integration boundaries — cash-in sweep, ledger shield/unshield, vault deposit/withdraw, bridge burn — so flagged depositors cannot use the privacy stack without making the protocol custodial. Also defines the public per-stealth depositor registry that lets cash-out paths re-evaluate the gate against the *current* state of every historical depositor (the deferred-discovery case) and the disposition routing per FHEIP-0010 reason code (OFAC refuses without refund; non-OFAC offers an opt-in paymaster-funded refund to the depositor; report-only allows + emits).
author: 0xOucan (@0xOucan)
discussions-to: TBD
status: Draft
type: Standards Track
category: ERC
created: 2026-04-28
requires: FHEIP-0010, FHEIP-0008, FHEIP-0003
---

## Abstract

A confidential FHE wallet stack (the encrypted ledger, the wrapper-token
shield/unshield, a confidential DeFi vault, a cross-chain bridge) all
share a structural invariant: there exist plaintext boundaries where
value enters or exits encryption. This FHEIP defines how those
boundaries consult an FHEIP-0010 compliance gate so flagged depositors
are refused at the boundary, and how a public per-stealth *depositor
registry* — populated at sweep time, read at every cash-out — lets the
boundary re-evaluate compliance against the *current* state of every
historical depositor (the deferred-discovery case where a depositor
was clean at deposit time but flagged hours later by a sanctions
oracle).

The design is strictly non-custodial. A gate rejection causes the
relevant entrypoint to revert; the protocol never holds, freezes, or
moves user funds. For non-OFAC reason codes the protocol MAY offer an
opt-in paymaster-funded refund-to-depositor (returning rejected funds
to the address that sent them); for OFAC-coded rejections (FHEIP-0010
code 1) the protocol MUST NOT refund — funds remain at the user's
cash-in stealth EOA.

This FHEIP bridges FHEIP-0010 (the gate interface) into the actual
operations a wallet stack performs, and supplies the data model
(depositor registry) the deferred-discovery enforcement needs.

## Motivation

FHEIP-0010 standardized the gate interface — `canShield`, `canUnshield`,
`ComplianceRejected(uint8)`, the reason-code registry. What it
deliberately did not standardize is *which call sites in a wallet stack
must invoke it*, *what they MUST do with rejections*, and *how
cash-out paths handle the case where a depositor was clean at deposit
time but is flagged at withdraw time*. Without that, every wallet
stack reinvents:

1. **Where to call the gate.** Some integrations only check at deposit;
   a flagged depositor's funds then exit cleanly at withdraw because no
   one re-evaluated. Other integrations re-check on every operation in
   the privacy stack and pay the gas. Both are reasonable; neither
   composes with another wallet's choice.

2. **How to reject.** Some integrations route flagged funds into a
   "compliance vault" the protocol controls — making the protocol a
   custodian, with all the regulatory weight that implies. Others
   refuse without refund. Others auto-return funds to the depositor —
   which is correct for non-OFAC rejections (FATF "rejected"
   semantics) but is *facilitating* for OFAC code 1 (FATF "blocked"
   semantics requires non-return). Mixing these is a regulatory
   landmine.

3. **How to remember who deposited.** A confidential ledger by design
   does not link cash-in stealth EOAs to the user's smart account on
   chain — that's the privacy property. But cash-out enforcement needs
   to enumerate the historical depositors that funded the user's
   commingled balance. Without a registry, the wallet has to derive
   that off-chain from `Transfer` events at runtime, which is slow and
   produces no audit trail.

This FHEIP standardizes all three, in a way that preserves the
non-custodial guarantee.

### Use cases

- A confidential vault (Tezcatli, FHE lending) plugged into a deny-list
  gate on a single chain, refusing OFAC-listed depositors at sweep and
  re-evaluating at every withdraw.
- A multi-chain confidential bridge that treats the bridge burn as a
  cash-out for compliance purposes — sending value to another chain
  in plaintext is functionally an exit from the privacy stack.
- A wallet UX showing "1 cash-in from 0xABC… is now flagged. Refund or
  hold?" with the hold path being the default safe choice.
- A regulator-facing audit trail reconstructing why a particular
  cash-out attempt was refused without subpoena access to the wallet's
  off-chain state.

### Non-goals

- Defining the gate's deny-list source (Chainalysis vs OFAC vs
  in-house) — that's FHEIP-0010 §4 plus governance.
- Standardizing identity/KYC verification flows — out of scope; the
  gate may consult an external supplier, but its on-chain interface is
  yes/no + reason code.
- Mandating that confidential vaults adopt this. Permissionless vaults
  with `complianceEnabled() == false` are outside this FHEIP's
  enforcement scope.
- Solving the internal-transfer taint propagation problem (a flagged
  user transferring to a clean user inside the privacy stack). That
  requires encrypted association proofs and is deferred to a future
  FHEIP.

## Specification

### 1. Architecture

The wallet stack is divided into **integration boundaries** —
entrypoints where plaintext value crosses into the encrypted layer or
exits from it — and **internal operations** that move only encrypted
state. Compliance enforcement applies at integration boundaries, not at
internal operations.

The integration boundaries this FHEIP recognizes:

| Boundary | Direction | Entrypoint shape |
|---|---|---|
| Sweep | Plaintext → encrypted | `privateSweepToLedger(funder, amount, …)` |
| Shield | Plaintext → encrypted | `Wrapper.shield(amount)` |
| Vault deposit | Encrypted → encrypted (via vault) | `Vault.depositConfidential(…)` |
| Vault withdraw | Encrypted → encrypted | `Vault.withdrawConfidential(…)` |
| Unshield | Encrypted → plaintext | `Wrapper.unshield(amount)` |
| Bridge burn | Encrypted → plaintext (cross-chain) | `Bridge.burnToBridge(…)` |

Sweep, Shield, and Vault deposit are **cash-in boundaries** — the gate
is consulted on the funder address. Withdraw, Unshield, and Bridge burn
are **cash-out boundaries** — the gate is consulted on every
historical depositor recorded in the depositor registry, plus the
destination address.

### 2. The depositor registry

```solidity
struct DepositRecord {
    address depositor;
    uint256 amount;
    uint64  timestamp;
    bytes32 sweepTxHash;
}

interface IZ0tzDepositorRegistry {
    /// @notice Append a record. Trusted-writers only (sweepers).
    function recordDeposit(
        address cashInStealth,
        address depositor,
        uint256 amount,
        bytes32 sweepTxHash
    ) external;

    function recordCount(address cashInStealth) external view returns (uint256);
    function recordAt(address cashInStealth, uint256 index) external view returns (DepositRecord memory);
    function recordsOf(address cashInStealth) external view returns (DepositRecord[] memory);
    function summaryOf(address cashInStealth)
        external view returns (uint256 totalAmount, uint256 numRecords);
}
```

**Privacy posture.** The `cashInStealth → depositor` link is already
public on-chain — it's the ERC-20 `Transfer(from=depositor,
to=cashInStealth, amount)` event the depositor's tx emits. The
registry is therefore an indexer over already-public data; it adds
zero new privacy regression. What the registry deliberately does NOT
store is the user's smart-account / ledger ID (which would link the
cash-in stealth back to the user's identity).

**Write authority.** Only registered writers (sweepers) may call
`recordDeposit`. The registry MUST gate this with an admin-controlled
`isWriter` mapping; the admin SHOULD be a multisig in production.

**Append-only.** Records MUST NOT be deleted or rewritten. A buggy
sweep that records a wrong depositor is fixed by a corrective
deployment, not by mutating history — the audit trail is more valuable
than the convenience of a delete.

### 3. Cash-in boundaries (sweep / shield / vault deposit)

```solidity
function privateSweepToLedger(address funder, uint256 amount, …) external {
    if (gate.complianceEnabled()) {
        (bool allowed, uint8 reason) = gate.canShield(funder, periodTag, amount);
        if (!allowed) revert ComplianceRejected(reason);
        if (reason == REPORT_REQUIRED) {
            emit ComplianceReportRequired(funder, periodTag, amount, keccak256("sweep"));
        }
    }
    // … existing sweep logic …
    registry.recordDeposit(cashInStealth, funder, amount, keccak256(blockhash(block.number-1), tx.hash));
    // … FHE-wrap into ledger …
}
```

The sweep MUST consult the gate before any state-changing operation,
MUST revert on rejection, and MUST record the deposit AFTER acceptance
but BEFORE the FHE-wrap step so a reverting wrap doesn't leave a
phantom registry entry.

### 4. Cash-out boundaries (unshield / vault withdraw / bridge burn)

```solidity
function unshield(uint256 amount, address destination) external {
    if (gate.complianceEnabled()) {
        // Re-evaluate every historical depositor against the CURRENT
        // gate state. This is the deferred-discovery enforcement.
        DepositRecord[] memory records = registry.recordsOf(cashInStealthOf(msg.sender));
        for (uint256 i; i < records.length; ++i) {
            (bool allowed, uint8 reason) = gate.canUnshield(records[i].depositor, periodTag, records[i].amount);
            if (!allowed) revert ComplianceRejected(reason);
            // REPORT_REQUIRED on a depositor surfaces but does not block.
            if (reason == REPORT_REQUIRED) {
                emit ComplianceReportRequired(records[i].depositor, periodTag, records[i].amount, keccak256("unshield"));
            }
        }
        // Also screen the destination.
        (bool dstAllowed, uint8 dstReason) = gate.canUnshield(destination, periodTag, amount);
        if (!dstAllowed) revert ComplianceRejected(dstReason);
    }
    // … existing unshield logic …
}
```

Cash-out boundaries MUST loop the depositor registry for the user's
cash-in stealth and call `gate.canUnshield(depositor_i, …)` for each
historical depositor. Any single rejection MUST cause the whole
operation to revert. The destination address (the address receiving
plaintext value or the cross-chain mint recipient for bridges) MUST
also be screened against the gate.

For bridge burns, the `destination` field is the remote-chain
`mintRecipient`, not the local destination — bridges treat this as the
exit address for compliance purposes.

### 5. Pro-rata attribution

A user's commingled balance may be drawn from multiple historical
depositors. When some are flagged at cash-out time and others are not,
the cash-out path MUST attribute amounts pro-rata across the depositor
set:

```
let total      = sum(records[i].amount for i in 0..n)
let cleanShare = sum(records[i].amount for i in 0..n if canUnshield(records[i].depositor) is allowed) / total
let exitable   = floor(cleanShare * requestedAmount)
let blocked    = requestedAmount - exitable
```

Implementations MAY refuse the entire operation when any depositor is
flagged (the simplest behavior and the one matching §4 above), or MAY
allow a partial exit of the `exitable` portion while leaving `blocked`
in the encrypted layer. If the partial-exit path is taken, the
implementation MUST clearly emit the breakdown:

```solidity
event PartialComplianceExit(
    address indexed user,
    uint256          requestedAmount,
    uint256          exitedAmount,
    uint256          blockedAmount
);
```

The all-or-nothing path is the conservative default. The partial-exit
path is operationally friendlier but adds attack surface (a malicious
sender depositing 1 wei from a flagged address to "poison" the
recipient's balance). Implementations choosing the partial-exit path
SHOULD impose a minimum-blocked-amount threshold below which the
poison case reverts.

### 6. Disposition routing per reason code

When `ComplianceRejected(reason)` is raised, the wallet (off-chain)
MUST route the response based on `reason`:

| Code | Mnemonic | On-chain | Off-chain disposition |
|---|---|---|---|
| 1 | `BLOCK_LIST` (OFAC) | Revert | NO refund — funds remain at user EOA. Wallet shows "Z0tz cannot process funds from this source. Consult a compliance professional." |
| 2 | `GEOFENCE` | Revert | Optional paymaster-funded refund to depositor. User MAY choose to leave funds at EOA. |
| 3 | `KYC_REQUIRED` | Revert | Same as 2. |
| 4 | `KYC_INSUFFICIENT_TIER` | Revert | Same as 2. |
| 5 | `PERIOD_CAP_EXCEEDED` | Revert | Wait + retry, OR refund, OR split below cap. |
| 6 | `AMOUNT_THRESHOLD` | Revert | Refund OR split below threshold. |
| 7 | `REPORT_REQUIRED` | Allow + emit | No rejection; `ComplianceReportRequired` event drives off-chain reporting. |

The OFAC distinction (code 1) is regulatorily load-bearing. Returning
sanctioned funds to the source address is *facilitating* the
sanctioned actor and is the primary reason a "compliance vault" path
fails the FATF "blocked" test. The pure-refusal path is the only
clean answer that preserves both the non-custodial guarantee and the
sanctions posture.

### 7. Audit-trail events

Every gate consultation MUST emit an audit event tying the decision to
a `uuid` and the gate's current `policyVersion`:

```solidity
event Screened(
    bytes32 indexed uuid,
    bytes32 indexed action,    // keccak256("sweep") | "unshield" | "bridge" | …
    address indexed account,   // depositor / funder / destination
    bytes32          periodTag,
    uint256          amount,
    bool             allowed,
    uint8            reasonCode,
    uint64           policyVersion
);
```

The event is emitted by the **integration entrypoint** (sweeper /
ledger / vault / bridge), not the gate itself. The gate's job is the
predicate; the entrypoint owns the audit trail.

`uuid` SHOULD be `keccak256(action || msg.sender || account || amount
|| block.number || tx.origin)` or a similar nonce that uniquely
identifies this screening within the chain history.

`policyVersion` is read from the gate; it lets historical
reconstruction answer "what rules were in effect when this user was
screened on date X" without trusting the gate's current state.

### 8. The IConfidentialOperationGated companion interface

Wallets preflighting via `eth_call` MUST be able to discover whether a
given contract is gated and which gate it consults:

```solidity
interface IConfidentialOperationGated {
    function complianceGate() external view returns (address);
    function complianceEnabled() external view returns (bool);
    function depositorRegistry() external view returns (address);
}
```

`complianceGate` and `complianceEnabled` are reused from FHEIP-0010 §4.
`depositorRegistry` is new — it lets a wallet read the historical
depositor set without the wallet having to know which registry the
contract uses.

## Rationale

**Why a public depositor registry instead of off-chain inference?** A
wallet could reconstruct the depositor set from `Transfer` events at
runtime, but: (a) it's slow (O(N) RPC calls per cash-out preflight),
(b) it has no audit-trail artifact for regulators, and (c) every
wallet would re-implement the same logic. The registry is a one-time
write at sweep time — the marginal cost is trivial and the wallet
contract simplifies to a single `recordsOf` read.

**Why pro-rata attribution and not FIFO/LIFO?** Pro-rata is the most
defensible accounting under FATF's source-of-funds doctrine: it
doesn't allow the user to selectively spend "old clean money before
new questionable money" via clever ordering. FIFO/LIFO are
operationally friendlier but invite accounting arbitrage. Pro-rata is
the safe default; implementations MAY expose FIFO as an opt-in flag,
but the default MUST be pro-rata.

**Why screen the destination at cash-out?** A determined bad actor
could try to "wash" funds by cashing out into another wallet they
control that's already on the deny-list — laundering the address but
not the funds. Screening the destination at cash-out closes that loop.

**Why no cash-out cap on REPORT_REQUIRED depositors?** Code 7
(REPORT_REQUIRED) is not a rejection — it's an "allowed but emit a
compliance event" outcome. A depositor flagged code 7 means downstream
reporting will happen; it does not mean the funds are blocked. Capping
or rejecting on code 7 would break the design: the entire point is
that code 7 is a non-blocking flag for FATF travel-rule-style scenarios.

**Why is the gate query at the integration entrypoint, not in the
gate?** Symmetry with FHEIP-0010 §4. The gate is a pure predicate
(`view`-only); the entrypoint owns the state mutation, the audit
emission, and the revert. Mixing those into the gate makes the gate a
non-`view` contract and breaks `eth_call` preflighting.

**Why no automatic refund of OFAC code 1?** Returning sanctioned funds
to the source address is *facilitating* the sanctioned actor under
OFAC's recordkeeping doctrine (see OFAC FAQ 1601 on blocked vs
rejected transactions). The clean answer is to reject the operation,
do nothing with the funds, and let the user (who controls the EOA
keys) deal with it through proper channels. This is the primary
reason this FHEIP differs from a "compliance vault" pattern: the
moment we route funds anywhere on a code-1 rejection, we trip the
custody / facilitating wire.

## Backwards Compatibility

Fully additive. Contracts that ship with `complianceGate() ==
address(0)` or `complianceEnabled() == false` behave exactly as before:
no gate query, no registry write, no audit event. Existing deployments
adopt this FHEIP by adding the views, wiring the gate consultation
into the entrypoints listed in §1, and registering the registry
writer.

The `Screened`, `ComplianceReportRequired`, and `PartialComplianceExit`
event signatures are new and do not collide with any existing standard
event. Wallet decoders that don't recognize them simply ignore them.

`ComplianceRejected(uint8)` is reused from FHEIP-0010 (selector
`0x4fc18568`); wallets that decoded that error from a vault now decode
it identically from a sweeper / bridge / ledger.

## Security Considerations

1. **Time-of-check / time-of-use.** Cash-out re-evaluation reads the
   gate state at submission time. Between an `eth_call` preflight and
   actual submission, the gate state can change (a new blacklist
   entry, a policy version bump). Implementations MUST re-call the
   gate inside the state-changing function and revert if the verdict
   has changed. Wallets MUST surface the possibility of preflight
   drift to the user.

2. **Registry poisoning.** A malicious sender depositing 1 wei from a
   flagged address to a target's cash-in stealth poisons the target's
   future cash-outs (since the registry will record a flagged
   depositor entry). Implementations choosing the all-or-nothing
   cash-out path SHOULD set a minimum-deposit threshold below which
   the registry write is skipped, OR offer a partial-exit path with a
   minimum-blocked-amount threshold (see §5). The trade-off is between
   poison resistance (high threshold) and audit completeness (low
   threshold).

3. **Registry write race.** Two sweeps targeting the same cash-in
   stealth in the same block could theoretically interleave their
   `recordDeposit` calls. The append-only design makes this safe —
   both records land, in some order — but consumers reading
   `recordsOf` MUST NOT assume any specific ordering beyond
   block-then-index.

4. **Gate availability.** A cash-out path that hard-requires the gate
   is bricked if the gate becomes uncallable (selfdestructed
   pre-EIP-6780, paused by upstream governance, etc.). Implementations
   SHOULD support `complianceEnabled() == false` as a graceful
   degradation path; a governance-controlled `setEnabled(false)` then
   gives the operator an emergency escape that keeps the cash-out
   path open while the gate is being repaired.

5. **Privacy leakage from the depositor registry.** The registry is
   public, but it contains data already public on-chain (`Transfer`
   events). The privacy property the wallet protects — that the user's
   smart-account / ledger ID is unlinkable from the cash-in stealth —
   is not affected, because the registry only knows
   `cashInStealth → depositor`, never `cashInStealth → user`.

6. **Reason-code substitution by a malicious gate.** A gate that
   returns `(true, 0)` for sanctioned addresses defeats the entire
   posture. The gate is a trusted oracle; integrators MUST audit the
   gate's deployer and code before binding. This FHEIP does not
   eliminate the trust requirement; it standardizes the channel.

7. **Cross-chain consistency.** Bridges treat the burn as a cash-out;
   the burn-side gate refuses if any historical depositor is flagged.
   The mint-side does NOT consult a gate (the funds have already
   exited the source-side privacy stack and arrive as plaintext). If
   the mint-side itself is a privacy stack, it consults its own gate
   on receipt as a cash-in.

8. **Registry size growth.** Long-lived cash-in stealths can accumulate
   many records. Implementations SHOULD impose a per-stealth record
   cap and either reject further deposits or rotate the cash-in
   stealth (the user generates a new derivation index from their
   passkey).

## Reference Implementation

- `Z0tzComplianceGate.sol` — Z0tz's deny-list gate implementing
  FHEIP-0010 §1, with KYC supplier integration via an `IKycRegistry`
  external contract and an `emitScreened` audit emitter (this repo's
  Z0tz fork at `contracts/contracts/compliance/`).
- `Z0tzDepositorRegistry.sol` — append-only registry implementing §2;
  trusted-writer access control, `summaryOf` aggregator (same path).
- `MockZ0tzKycRegistry.sol` — yes/no KYC supplier mock that bridges
  to external providers like Sumsub, Persona, or Chainalysis KYT in
  production.
- `Z0tz/cli/src/commands/compliance.ts` — admin CLI for the gate
  (`blacklist`, `whitelist`, `bulk`, `list`, `status`, `check`,
  `set-enabled`, `set-require-kyc`, `set-kyc-registry`).

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
