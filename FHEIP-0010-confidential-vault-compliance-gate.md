---
fheip: 0010
title: Confidential vault compliance gate interface
description: Standardize an opt-in `IConfidentialVaultComplianceGate.canShield/canUnshield(address, bytes32, uint256) returns (bool, uint8)` interface, the `ComplianceRejected(uint8)` typed revert, and a reason-code registry — so any FHE-encrypted vault can plug a regulator-facing AML/KYC oracle in front of shield/unshield without hardcoding a specific compliance vendor, and so wallets can preflight via `eth_call` and surface human-legible rejection reasons.
author: 0xOucan (@0xOucan)
discussions-to: TBD
status: Draft
type: Standards Track
category: ERC
created: 2026-04-23
requires: FHERC-20 (fhenix-confidential-contracts), FHEIP-0003, FHEIP-0008
---

## Abstract

Confidential FHE vaults (USDC → tzcUSDC, generic FHERC-20 wrapped strategies) sit at the boundary between plaintext value entering the vault and FHE-encrypted positions held inside it. Regulators increasingly expect that boundary to be a controllable choke point: deposits must be screenable against block-lists, geofences, and per-period caps; withdraws must be auditable; "report-but-allow" outcomes must exist. Today every confidential-vault deployment hardcodes a vendor — Chainalysis, an in-house gate, a mock — and clients reverse-engineer the rejection format per vault.

This FHEIP defines a vendor-neutral compliance gate interface that any vault MAY consult on shield / unshield, a typed `ComplianceRejected(uint8)` error so wallets can render human-legible reasons, and the registry of canonical reason codes (block-list, geofence, KYC tier, period cap, …). It is opt-in: a vault running with `gate == address(0)` is unaffected. The gate is `view`-only so wallets can preflight with `eth_call` and bail before a single wei of gas is burned.

## Motivation

The Tezcatli Confidential Vault on arb-sepolia ships with `MockComplianceGate` for testnet — admin-toggleable allow / deny. Z0tz's DeFi tab integrates with it to demo the full deposit / withdraw flow under realistic compliance constraints. Several pain points emerged:

1. **Selector-only failures.** A denied deposit reverts with `0x4fc18568` and no reason. Wallets have to maintain per-vault selector tables to render anything friendlier than "tx reverted".
2. **No preflight.** Without a `view` checker, the wallet either submits the tx blind (user pays gas to discover rejection) or duplicates the gate's logic off-chain (forks every time the gate updates).
3. **Vendor lock-in.** A vault wired against `IChainalysisOracle` can't swap to an in-house gate without redeploying. Conversely, a wallet wired to one vendor's API can't talk to a vault using a different one.
4. **No standard reason taxonomy.** "Geofence" and "KYC required" are different operational outcomes with different remediation paths, but every gate encodes them differently — sometimes as different revert strings, sometimes as different error selectors, sometimes as a numeric code with no published meaning.
5. **Report-but-allow gap.** Regulated counterparties often need "this tx is allowed but emits a compliance report" semantics (e.g., FATF travel-rule). Boolean-only gates can't express it.

This FHEIP collapses those into one interface the wallet, the vault, and the gate all agree on.

### Use cases

- Confidential vaults with optional regulator integration (Tezcatli, future tzc-* vaults, FHE lending pools)
- Multi-jurisdiction deployments that need different gates per vault (US-only, EU-only, permissionless)
- Wallet UX showing "deposit blocked: KYC tier 2 required" instead of "tx reverted (0x4fc18568)"
- Aggregator dashboards that scan a chain for vaults and surface compliance posture (gate address, gate type) per vault
- Audit trails reconstructing why a particular deposit attempt was rejected without reading off-chain logs

### Non-goals

- Defining HOW a gate decides to allow / deny. The gate is a black box; this FHEIP only standardizes the question and answer shapes.
- Mandating that vaults use a gate. Permissionless confidential vaults SHOULD continue to ship with `gate == address(0)`.
- Specifying KYC verification flows, identity-attestation formats, or off-chain attestation schemes.
- Defining the on-chain registry of approved compliance providers. That's governance, not protocol.

## Specification

### 1. The compliance gate interface

```solidity
interface IConfidentialVaultComplianceGate {
    /// @notice Decide whether `account` is allowed to shield `amount` underlying
    ///         into a confidential wrapper, optionally tagged with `periodTag`.
    /// @param  account     The depositor address (typically a stealth EOA).
    /// @param  periodTag   Opaque 32-byte scope key — gate-defined. Common values:
    ///                     keccak256("daily"), keccak256("monthly"), bytes32(0).
    /// @param  amount      Amount of underlying being shielded, in underlying decimals.
    /// @return allowed     True if the operation MAY proceed.
    /// @return reasonCode  Canonical reason code (see §3). MUST be 0 when allowed=true.
    function canShield(address account, bytes32 periodTag, uint256 amount)
        external view returns (bool allowed, uint8 reasonCode);

    /// @notice Decide whether `account` is allowed to unshield. Same semantics.
    function canUnshield(address account, bytes32 periodTag, uint256 amount)
        external view returns (bool allowed, uint8 reasonCode);
}
```

Both functions MUST be `view`. A gate that needs to mutate (e.g., to record a usage counter for per-period caps) MUST implement that mutation in a separate call hook (out of scope for this FHEIP).

### 2. The `ComplianceRejected` typed revert

```solidity
error ComplianceRejected(uint8 reasonCode);
```

When a vault's shield / unshield is gated and the gate returns `(false, code)`, the vault MUST revert with `ComplianceRejected(code)`. The vault MUST NOT swallow the rejection or substitute a generic error.

When the gate returns `(true, 7)` — see §3 — the vault MUST proceed AND emit:

```solidity
event ComplianceReportRequired(
    address indexed account,
    bytes32 indexed periodTag,
    uint256          amount,
    bytes32          op           // keccak256("shield") | keccak256("unshield")
);
```

The vault MUST NOT block the operation in this case; the event is for off-chain compliance recording only.

### 3. Reason-code registry

| Code | Mnemonic | Semantics |
|---|---|---|
| 0 | `OK` | Reserved. Returned with `allowed=true`. MUST NOT appear with `allowed=false`. |
| 1 | `BLOCK_LIST` | Account is on a sanctions / OFAC / internal block-list. Permanent until governance lifts it. |
| 2 | `GEOFENCE` | Account's last known jurisdiction is excluded from this vault. Remediation: re-attest from an allowed jurisdiction. |
| 3 | `KYC_REQUIRED` | Account has no KYC attestation on file. Remediation: complete KYC with the gate's attestor. |
| 4 | `KYC_INSUFFICIENT_TIER` | Account has KYC but at a tier below this vault's minimum. Remediation: upgrade tier. |
| 5 | `PERIOD_CAP_EXCEEDED` | This call would exceed the per-period cap (e.g., daily / monthly limit). Remediation: wait or split. |
| 6 | `AMOUNT_THRESHOLD` | Amount exceeds a hard absolute threshold for this account / vault combination. |
| 7 | `REPORT_REQUIRED` | Allowed but compliance report MUST be emitted. Vault MUST emit `ComplianceReportRequired` and proceed. |
| 8–127 | Reserved for future canonical codes. |
| 128–255 | Gate-defined. Wallets MUST render as "compliance check failed" with the raw code. |

Codes 1–6 are rejection codes (the gate returns `allowed=false`). Code 7 is the unique allow-with-report code. Codes 8–127 are reserved for this FHEIP's future revisions; gate authors MUST NOT use them. Codes 128–255 may be defined per gate for vendor-specific reasons; wallets that don't recognize them SHOULD show a generic message and the code.

### 4. The vault's gate-binding surface

```solidity
interface IConfidentialVaultGated {
    /// @notice The active compliance gate, or address(0) if no gate is configured.
    function complianceGate() external view returns (address);

    /// @notice True if the gate's verdict is currently enforced.
    /// @dev    A vault MAY have a gate configured but enforcement disabled (e.g.,
    ///         testnet pause, governance kill-switch). Wallets MUST consult both
    ///         `complianceGate()` and `complianceEnabled()` before preflighting.
    function complianceEnabled() external view returns (bool);
}
```

When `complianceEnabled() == false` OR `complianceGate() == address(0)`, the vault MUST behave as if no gate is present — `shield` / `unshield` SHALL NOT call the gate and MUST NOT revert with `ComplianceRejected`.

### 5. Wallet preflight pattern

A wallet preparing a deposit / withdraw SHOULD execute the following sequence before submitting any state-changing transaction:

```typescript
// 1. Discover gate posture.
const gate    = await vault.complianceGate();
const enabled = await vault.complianceEnabled();
if (gate === ZERO_ADDRESS || !enabled) {
    // No preflight needed. Submit the tx.
    return submit();
}

// 2. Preflight via eth_call.
const [allowed, code] = await IComplianceGate(gate).canShield(
    stealth, periodTag, amount,
);

if (!allowed) {
    throw new ComplianceError(code);   // surfaced as friendly text, see §6
}
if (code === 7) {
    // Allow with report. Tx will succeed; warn user that a compliance event
    // will be emitted.
}
return submit();
```

Wallets MUST treat the preflight as advisory: chain state may change between preflight and submission (a block-list update, a period-cap rollover). The vault remains the source of truth at execution time.

### 6. Wallet rendering convention

For codes 1–7, wallets SHOULD use the following human-legible strings (or their localized equivalents):

| Code | Default string |
|---|---|
| 1 | "Deposit blocked: account is on a sanctions list." |
| 2 | "Deposit blocked: this vault is not available in your jurisdiction." |
| 3 | "KYC required to deposit into this vault." |
| 4 | "Higher KYC tier required for this vault." |
| 5 | "Period cap exceeded — try again later or with a smaller amount." |
| 6 | "Amount exceeds this vault's per-tx threshold." |
| 7 | "Allowed — a compliance report will be emitted on submission." |

Wallets MAY include a "learn more" link to documentation specifying the gate vendor and remediation flow.

## Rationale

**Why view-only?** A non-view gate forces every preflight to either burn gas (via `eth_call` against a stateful function — still works, but the gate then can't reliably write) or duplicate the gate's logic off-chain. View-only gates compose cleanly with `eth_call`-based preflights and let the wallet show a fast yes/no without trusting an off-chain shadow of the gate. State writes (e.g., usage counters) belong in a separate `recordShield(account, ...)` hook called from the vault during execution, not in the predicate path.

**Why `(bool, uint8)` instead of `revert(reason)`?** A view function can't communicate via a typed revert without changing the call shape (the caller has to use `try/catch`). Returning `(bool, code)` is simpler, gas-cheaper for the happy path, and lets the gate explain itself even when allowing (code 7).

**Why a typed `ComplianceRejected(uint8)` instead of separate selectors per code?** Single selector + numeric code is friendlier for ABI decoders, lets future codes be added without proliferating selectors, and matches the FHEIP-0008 convention of one typed error covering a parameterized failure mode.

**Why `bytes32 periodTag` and not enum?** Period tags are gate-defined. Some gates have no period concept (always pass `bytes32(0)`); others have daily / weekly / monthly / per-tx scopes. An opaque tag is forward-compatible with new period semantics without versioning the interface.

**Why reserve code 0 for OK and not for "unspecified"?** Off-chain consumers should never see `(allowed=false, code=0)` because that's ambiguous about why. Forcing OK into 0 makes "code != 0" mean "something is being said about this call" and lets wallets handle the report path uniformly with the rejection path.

**Why is allowed-with-report code 7 and not in the rejection range?** Code 7 is a non-rejecting outcome — the tx will succeed. Putting it adjacent to the rejection codes lets it occupy the same `uint8` channel without an additional return value. Wallets check `allowed == false` for the rejection branch and `code == 7 && allowed == true` for the report branch.

**Why include `amount` in the gate query when the vault could pass it implicitly?** Period caps and amount thresholds need it. `account + periodTag + amount` is the minimal triple that covers all canonical codes; gates that don't use `amount` simply ignore it.

**Why `complianceEnabled()` separate from `complianceGate()`?** Operational kill-switch. A vault deployed against a gate may need to disable enforcement temporarily (gate bug, mainnet incident, governance pause) without wiping the gate address. Two views distinguish "no gate configured" from "gate paused" cleanly.

## Backwards Compatibility

Fully additive. Vaults already deployed without a gate continue to work with `complianceGate() == address(0)` (or by not implementing the views, in which case wallets default to "no preflight"). The Tezcatli vault on arb-sepolia already implements this surface; existing FHE vault deployments can adopt by adding the views and wiring the gate consultation into shield / unshield with no caller-facing change for unconfigured deployments.

The `ComplianceRejected(uint8)` selector is `0x4fc18568` (already shipped by Tezcatli). New gate-bearing vaults SHOULD reuse this selector to share wallet decoders.

## Security Considerations

1. **Time-of-check / time-of-use.** Preflight via `eth_call` does not bind the gate's verdict to the eventual transaction. A reorg, a block-list update, or a period-cap rollover between preflight and execution can flip the answer. Vaults MUST re-call the gate inside the state-changing function and revert if the verdict has changed. Wallets MUST surface this possibility (e.g., "the gate may re-evaluate at submission").
2. **Gate availability.** A vault that hard-depends on a gate is bricked if the gate becomes uncallable (selfdestructed in pre-EIP-6780, paused by upstream governance, etc.). Vaults SHOULD support `complianceEnabled() == false` as a graceful degradation path. Hard-coupled vaults SHOULD design an emergency escape (governance-controlled `setGate`) before mainnet.
3. **Reason-code substitution.** A malicious gate could lie — return `(true, 0)` for sanctioned addresses, or `(false, 1)` for non-sanctioned addresses. The gate is a trusted oracle; vault deployers MUST audit gate code before binding. This FHEIP does not eliminate the trust requirement; it only standardizes the channel.
4. **Privacy leakage to the gate.** The gate sees `(account, amount, op)` per call. For Z0tz this is a stealth EOA + plaintext underlying amount; the gate cannot link to the user's smart account or ledger. But a gate that logs queries off-chain creates a deanonymization sidechannel. Wallets MUST disclose the gate vendor to the user before first use; users can then decide whether the operational privacy posture matches their threat model.
5. **Reason-code privacy.** Code `KYC_INSUFFICIENT_TIER` reveals to anyone watching `eth_call` traffic that this account has *some* KYC on file. Most off-chain threat models accept this; high-privacy threat models may want gates that downgrade specific codes to a generic "rejected" before returning.
6. **Replay across vaults.** Two vaults sharing one gate but with different period caps could see correlated denials. Per-vault gate instances avoid this; shared gates SHOULD scope their internal accounting by `(vault, account, periodTag)` rather than just `(account, periodTag)`.
7. **Code 7 spoofing.** A gate that always returns `(true, 7)` would force every operation to emit `ComplianceReportRequired`, bloating event volume. This is detectable and not a security issue per se, but vaults SHOULD cap event emissions or charge for them if abuse appears.
8. **Off-chain rendering injection.** Wallets rendering `code` text MUST treat the message as static (looked up from the rendering convention table), not a string returned by the gate. A gate that could return arbitrary text would be a phishing vector.

## Reference Implementation

- Tezcatli's gate interface (deployed on arb-sepolia): the `ITezcatliVaultComplianceGate` referenced from `Z0tz/cli/src/core/defi.ts` matches §1 with `(bool, uint8)` returns. The `ComplianceRejected(uint8)` selector `0x4fc18568` is reused verbatim.
- Z0tz's preflight pattern: [`Z0tz/gui/src/main/ipc-handlers.ts`](https://github.com/0xOucan/Z0tz/blob/DeFi/gui/src/main/ipc-handlers.ts) — `preflightVaultCompliance(pub, vault, stealth, amount, kind)` does the §5 sequence before submitting any DeFi state-changing tx.
- Z0tz's friendly-error mapping: `explainDefiError(err)` in the same file decodes `ComplianceRejected(N)` into the §6 strings.
- MockComplianceGate (testnet): admin-toggleable boolean per direction, returns code 0 or 1. Useful as a starting skeleton for new gates.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
