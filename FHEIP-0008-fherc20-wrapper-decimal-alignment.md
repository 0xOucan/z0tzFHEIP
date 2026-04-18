---
fheip: 0008
title: FHERC-20 wrapper decimal-alignment standard
description: Standardize the ERC-20 ↔ FHERC-20 wrapper conversion: expose `conversionRate()` and `wrapperDecimals()` views, emit `WrapDustBurned` when sub-precision dust is forfeited, and revert with a typed `AmountBelowPrecision` error on shield amounts below the conversion rate — so wallets and indexers can handle decimal mismatches uniformly.
author: 0xOucan (@0xOucan)
discussions-to: TBD
status: Draft
type: Standards Track
category: ERC
created: 2026-04-17
requires: FHERC-20 (fhenix-confidential-contracts), FHEIP-0003
---

## Abstract

`euint64` caps at 2^64 − 1 (~1.8 × 10^19). ERC-20 tokens routinely use 18 decimals, blowing past that cap long before hitting realistic token supplies. `FHERC20WrappedERC20` addresses this by silently truncating the underlying's decimals to 6 (or similar) via a `conversionRate` multiplier — but the behavior is unadvertised: wallets see the wrapper claim `decimals() == 6`, users deposit 18-decimal amounts assuming parity, and sub-precision dust is burned with no event. This FHEIP formalizes the wrapping contract's interface: one view for the conversion rate, one view for the cap, one event for dust burns, and one typed error for rejected shields. Indexers and wallets can then display "you wrapped 1.234567 TOKEN (0.000000001234 dust forfeited)" without per-wrapper bespoke parsing.

## Motivation

Three footguns today:

### 1. Silent dust burning

`shield(amount)` with `amount < conversionRate` either reverts with an opaque error (`AmountTooSmallForConfidentialPrecision`) or — depending on implementation — silently drops the sub-precision remainder. A user shielding 1.234567890123456789 TOKEN into a 6-decimal wrapper loses the 12 sub-digits with no on-chain record.

### 2. Wallets can't render the actual cap

A wallet showing "max shieldable: 2^64 / conversionRate" has to know `conversionRate` — but the current FHERC-20 wrapper doesn't expose it as a standard view. Every wallet reverse-engineers it from constructor arguments or documentation.

### 3. No uniform precision error

Different wrappers revert with different error strings or even succeed-with-zero. Wallets can't show "this amount is below the wrapper's precision" cleanly.

### Use cases

- Wallet UI rendering max-wrappable / dust-forfeit warnings
- Multi-wrapper dashboards (one UI for USDC, USDT, DAI, WETH wrappers)
- Audit tools reconciling wrapped supply ↔ underlying supply
- Indexers producing "effective supply" figures

### Non-goals

- Changing `euint64`'s bit width.
- Mandating a specific `wrapperDecimals` value — that stays up to the wrapper author.
- Defining wrapping semantics for non-decimal-denominated underlyings (NFTs, rebasing tokens).

## Specification

### 1. Required views

```solidity
interface IFHERC20Wrapper {
    /// @notice Decimals of the FHERC-20 wrapper. MUST be ≤ 8 for euint64 safety.
    function wrapperDecimals() external view returns (uint8);

    /// @notice Multiplier from wrapper-units to underlying-units.
    /// @dev    underlying = wrapped * conversionRate
    /// @dev    MUST equal 10 ** (underlyingDecimals - wrapperDecimals)
    function conversionRate() external view returns (uint256);

    /// @notice Address of the underlying ERC-20.
    function underlying() external view returns (address);
}
```

- `wrapperDecimals() ≤ 8`. Wrappers MAY choose smaller values (e.g., 6 for USDC-style) but MUST NOT exceed 8 because euint64's max is ~1.8e19 and we reserve ≥2 decimal digits for supply safety margin.
- `conversionRate() = 10**(underlyingDecimals - wrapperDecimals)`. If `underlyingDecimals == wrapperDecimals`, conversionRate is `1`.
- These views MUST be pure-view (no storage writes).

### 2. Dust-burn event

```solidity
event WrapDustBurned(
    address indexed user,
    uint256           underlyingAmountAttempted,
    uint256           dustAmount,
    uint256           wrappedAmount
);
```

Emitted by `shield(amount)` when `amount % conversionRate != 0`. The emitting wrapper MUST either (a) refund `dustAmount` back to `user`, or (b) keep it in the wrapper's underlying balance and emit `WrapDustBurned` for transparency. Implementations SHOULD refund; if they don't, they MUST emit.

### 3. Typed rejection error

```solidity
error AmountBelowPrecision(uint256 providedAmount, uint256 minAmount);
```

`shield(amount)` with `amount < conversionRate` MUST revert with `AmountBelowPrecision(amount, conversionRate)`. This is a typed custom error (EIP-838) so wallets can catch and render uniformly.

### 4. Unshield symmetry

`unshield(wrappedAmount)` returns exactly `wrappedAmount * conversionRate` to the user. No sub-precision loss on the unshield path (because the wrapper is denominated in wrapper-units, not underlying-units). Implementations MUST document this explicitly in NatSpec.

### 5. FHEIP-0003 event integration

When a wrapper emits `WrapDustBurned`, it SHOULD also emit the corresponding FHEIP-0003 event:

```solidity
// Shield = burn underlying, mint wrapped
ConfidentialCredit(accountKey=user, source=bytes32(0), encAmount=<wrapped ciphertext>, op=0);
WrapDustBurned(user, underlyingAmountAttempted=amount, dustAmount, wrappedAmount);
```

This lets a single scanner reconstruct both sides of the wrap.

## Rationale

**Why cap `wrapperDecimals` at 8?** `2^64 - 1 ≈ 1.8 × 10^19`. With 8 decimals we reserve 10^19 / 10^8 ≈ 10^11 units of supply — 100 billion wrapped tokens, comfortable for most real-world supplies (USDC circulating supply is ~35 billion). 18 decimals would leave only 18 units of supply, unusable.

**Why `conversionRate` as `uint256` and not a ratio tuple?** Single-value storage is cheaper and simpler. The rate is always a power of 10 in practice, so a tuple adds no flexibility.

**Why require emitting even if refunded?** Refund is strictly better UX than burning, but off-chain tooling still wants to see "user attempted to shield 1.23456789 but only 1.234567 went through" for confirmation flows. The event is cheap (indexed + 3 non-indexed fields) and diagnostics-valuable.

**Why a typed error instead of a revert string?** EIP-838 errors are cheaper (no string storage in code), parseable in ABI decoders, and let wallets render localized messages. Revert strings are discouraged in modern Solidity for exactly these reasons.

**Why not just require `wrapperDecimals() == underlying.decimals()` always?** That would limit FHERC-20 wrappers to tokens with ≤8 decimals, excluding DAI (18), WETH (18), and most DeFi tokens. The decimal mismatch is fundamental to making euint64 usable.

## Backwards Compatibility

Additive for wrappers not yet implementing these views — adding the view functions and one event is a non-breaking upgrade. Wrappers currently using opaque reverts (`"AmountTooSmallForConfidentialPrecision"`) can switch to `AmountBelowPrecision` in a minor version; revert-string callers that were catching by hash remain broken either way.

Wallets that currently read `decimals()` continue to work — this FHEIP adds `wrapperDecimals()` as an explicit second view rather than overloading `decimals()`. (`decimals()` SHOULD continue to return `wrapperDecimals()` for ERC-20 UI compatibility.)

## Security Considerations

1. **Dust accumulation as protocol fee.** A wrapper that silently keeps dust builds up a balance in the underlying token. Over time this becomes a material sum. Wrappers SHOULD either refund dust per-tx (clean but slightly more gas) or publish the dust balance as a protocol fee with clear governance around who controls it. Silent retention without disclosure is a rug-pull vector.
2. **ConversionRate immutability.** `conversionRate` MUST be set at construction and MUST NOT be mutable post-deployment. A mutable conversion rate would retroactively change the meaning of every wrapped balance — catastrophic for holders.
3. **`underlyingDecimals` drift.** If the underlying ERC-20's `decimals()` is somehow mutable (extremely unusual, but possible with upgradeable proxies), the wrapper's `conversionRate` becomes wrong. Wrappers MUST read and freeze `underlyingDecimals` at construction from the underlying; they MUST NOT re-query on every operation.
4. **Max-supply overflow.** Minting more wrapped tokens than `(2^64 - 1) / conversionRate` wraps to 0 silently under FHE arithmetic. Wrappers MUST enforce a supply cap either by tracking shielded total (in plaintext uint256) and rejecting mints that would exceed the euint64 boundary, or by inheriting an explicit cap pattern from OpenZeppelin.
5. **Event-ordering across FHEIP-0003.** Indexers reconstructing "wrap" flows expect `ConfidentialCredit` and `WrapDustBurned` in the same tx. Wrappers MUST emit both events in the same tx (`shield`); split emissions across multiple txs break reconstruction.

## Reference Implementation

- Current wrapper with decimals hard-coded: `fhenix-confidential-contracts/contracts/FHERC20WrappedERC20.sol` — constructor at lines 51-53 caps at 6 via `decimals() <= 6 ? decimals() : 6`. The conversion rate is computed internally but never exposed as a public view.
- Z0tz's manual decimal check: [`Z0tzPrivateLedgerVault.sol` lines 102-104](https://github.com/0xOucan/Z0tz/blob/main/contracts/contracts/ledger/Z0tzPrivateLedgerVault.sol#L102-L104) — `uint8 d = IERC20Metadata(_underlying).decimals(); if (d > 8) revert DecimalsTooHigh();` — the typed error matches this FHEIP's `AmountBelowPrecision` convention.
- Example of typed-error migration in OpenZeppelin: `@openzeppelin/contracts/interfaces/draft-IERC6093.sol` (custom-error conventions for ERC-20).

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
