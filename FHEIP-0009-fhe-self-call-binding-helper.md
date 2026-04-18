---
fheip: 0009
title: Canonical self-call binding helper for relayer-submitted FHE ops
description: Add `FHE.asEuintNSelf(input)` as a canonical primitive that binds the ACL of a relayer-submitted `InEuintN` to the consuming contract rather than to `msg.sender`, replacing the ad-hoc "`this.verifyAmount()` external self-call" pattern every multi-contract FHE system reinvents.
author: 0xOucan (@0xOucan)
discussions-to: TBD
status: Draft
type: Standards Track
category: Core
created: 2026-04-17
requires: FHE.sol (cofhe-contracts), FHEIP-0004
---

## Abstract

`FHE.asEuintN(InEuintN calldata input)` converts an externally-provided ciphertext into an internal handle and binds ACL to `msg.sender`. When a contract is called by a relayer and needs the handle bound to itself (not to the relayer), the author must write an external-visible `verifyAmount` helper and invoke it via `this.verifyAmount(input)` — an external self-call that flips `msg.sender` to the contract's own address. This pattern is correct but entirely ceremonial: every author rediscovers it, every author writes the same 6-line helper, and miss-auditing it leaks ACL to the wrong address. This FHEIP adds `FHE.asEuintNSelf(input)` as a single-call primitive with the same semantics, removing the boilerplate.

## Motivation

### The pattern today

```solidity
contract Ledger {
    function spend(InEuint64 calldata amount, /* ... */) external {
        // msg.sender == relayer; FHE.asEuint64 would bind ACL to the relayer.
        // We need ACL bound to address(this). Force a self-call:
        euint64 a = this.verifyAmount(amount);
        // ... use a ...
    }

    function verifyAmount(InEuint64 calldata input) external returns (euint64) {
        if (msg.sender != address(this)) revert NotSelf();
        return FHE.asEuint64(input);
    }
}
```

Every relayer-served FHE contract needs this. Z0tz's ledger has it. Future FHE lending markets, prediction markets, and AMMs with gas-sponsorship will all need it. The risks of reinvention:

- **Miss the `msg.sender != address(this)` guard.** An attacker calls `verifyAmount` directly, binding ACL to *themselves* instead of the contract. Now the attacker can decrypt the ciphertext.
- **Miss the `external` visibility.** An internal `verifyAmount` doesn't flip `msg.sender` — the whole pattern silently fails, ACL still binds to the relayer.
- **Gas waste.** External self-calls cost ~700 extra gas per invocation. Not huge, but not zero, and entirely avoidable.
- **Readability.** The pattern is obfuscated: a reader seeing `this.verifyAmount(input)` has to know the trick. A reader seeing `FHE.asEuint64Self(input)` does not.

### Use cases

- ERC-4337 UserOp-relayed FHE operations
- Gasless meta-transactions
- Multi-contract FHE systems where one contract calls another to perform an FHE op on behalf of a user
- Any FHE contract that ever sees `msg.sender != tx.origin`

### Non-goals

- Changing `FHE.asEuintN`'s default binding (remains `msg.sender`).
- Introducing a way to bind ACL to an arbitrary third-party address (that would be a different primitive with different security properties).

## Specification

### 1. Primitive

```solidity
library FHE {
    /// @notice Converts `InEuint64` to `euint64` with ACL bound to `address(this)`
    ///         rather than to `msg.sender`. Semantically equivalent to:
    ///             `this.verifyAmountExternal(input)` where
    ///             `verifyAmountExternal` calls `FHE.asEuint64(input)`.
    /// @dev    Implementations MAY fuse the self-call into a single internal
    ///         dispatch via a precompile or system-level context switch; in
    ///         either case the observable ACL state MUST be identical to the
    ///         external-self-call pattern.
    function asEuint64Self(InEuint64 calldata input) internal returns (euint64);
    function asEuint8Self(InEuint8 calldata input) internal returns (euint8);
    function asEuint16Self(InEuint16 calldata input) internal returns (euint16);
    function asEuint32Self(InEuint32 calldata input) internal returns (euint32);
    function asEuint128Self(InEuint128 calldata input) internal returns (euint128);
    function asEboolSelf(InEbool calldata input) internal returns (ebool);
    function asEaddressSelf(InEaddress calldata input) internal returns (eaddress);
}
```

Normative behavior:

- Post-call ACL state MUST be identical to the state after the manual `this.verifyAmountExternal(input)` pattern where `verifyAmountExternal` is guarded by `msg.sender == address(this)` and returns `FHE.asEuintN(input)`.
- `asEuintNSelf` MUST NOT cost more gas than the manual pattern. Compliant implementations MAY fuse via a context-switch precompile; baseline implementations MAY simply inline the external-self-call.
- The caller's transient ACL on the returned handle is granted to `address(this)`, not to `msg.sender`.

### 2. Documentation requirement

The NatSpec on every `FHE.asEuintN` variant MUST include a pointer to its `Self`-suffixed counterpart and explain the binding distinction:

```solidity
/// @notice Converts `InEuint64` to `euint64` with ACL bound to `msg.sender`.
/// @dev    For contracts called by relayers where the ACL should bind to the
///         contract itself (not to the relayer), use `FHE.asEuint64Self(input)`.
///         See FHEIP-0009 for the binding-attack background.
function asEuint64(InEuint64 calldata input) internal returns (euint64);
```

### 3. Static-analysis signal

Tooling (solhint plugin, slither detector) SHOULD flag any function that:

- accepts `InEuintN calldata` as a parameter, AND
- calls `FHE.asEuintN` directly, AND
- does NOT match a "self-only" guard pattern (either `require(msg.sender == address(this))` or the function is external and only called via `this.functionName(...)`).

This catches both legitimate uses of the manual pattern (and suggests migration to `asEuintNSelf`) and misuses where a contract author forgot to self-call and silently leaked ACL to the relayer.

## Rationale

**Why a new primitive instead of just documenting the pattern?** Documentation is necessary but not sufficient — FHEIP-0004 and FHEIP-0007 both point out places where docs lag behind observed practice. A primitive is impossible to forget; a doc is easy to miss. Making the safe path the shortest path aligns with the least-surprise principle.

**Why not change the default of `asEuintN` to bind to `address(this)`?** Backwards compatibility with existing direct-user-call flows. A user calling `ledger.register(InEuint64(...))` directly expects ACL to bind to themselves (the user), not to the ledger. Changing the default would break that model. The `Self`-suffix is a targeted addition, not a rebinding of existing behavior.

**Why include `asEboolSelf` and `asEaddressSelf` even though relayer-submitted booleans and addresses are rare?** Uniformity. If the primitive exists for all FHE types, static analysis is simpler (one rule: "flag any `asEuintN` in a relayer context that's not the Self variant"). Scoped-to-uint primitives would require type-aware rules.

**Why forbid binding to arbitrary third-party addresses?** That would be a different primitive — essentially letting a user say "give ACL to address X, not me, not this contract." Useful, but opens broader attack surface (relayer substituting X with a malicious contract), and better handled via explicit permit flows (FHEIP-0006) rather than an input-coercion primitive.

## Backwards Compatibility

Fully additive. Existing contracts using `this.verifyAmount(input)` continue to work exactly as before — the self-call pattern is still legal. `asEuintNSelf` is a shortcut. Migration is opt-in per contract; no forced upgrades.

## Security Considerations

1. **Semantic equivalence is the spec.** An implementation that does something subtly different from the manual pattern (e.g., binds ACL to an intermediate proxy address) violates the spec even if it appears to work in simple cases. Compliance tests MUST check that ACL state matches byte-for-byte.
2. **Context-switch precompile risks.** If an implementation fuses the self-call into a precompile that changes `msg.sender` internally, that precompile's implementation is security-critical. A bug in the precompile that binds ACL to the wrong address is a catastrophic confidential-data leak.
3. **Don't use `asEuintNSelf` in direct-user paths.** If a user directly calls `contract.foo(InEuint64(...))` and `foo` uses `asEuintNSelf`, ACL binds to the contract, not the user. The user then cannot decrypt their own input handle off-chain without an explicit `FHE.allow(handle, userAddress)` inside `foo`. This is sometimes desired (the contract wants sole custody of the handle) and sometimes not — contract authors must be explicit.
4. **Composition with FHEIP-0007.** A contract using `asEuintNSelf` still needs `FHE.bindCtHash` in its authorization digest to prevent relayer substitution. The two FHEIPs solve orthogonal problems: 0007 prevents the relayer from choosing which ciphertext, 0009 ensures the ciphertext's ACL is bound to the right consumer.
5. **Gas parity.** Implementations that are slower than the manual pattern create an economic incentive to keep the manual pattern, undermining the migration story. The spec's "MUST NOT cost more gas" is a hard requirement, not aspirational.

## Reference Implementation

- Z0tz's manual self-call pattern: [`Z0tzPrivateLedger.sol` lines 303-306](https://github.com/0xOucan/Z0tz/blob/main/contracts/contracts/ledger/Z0tzPrivateLedger.sol#L303-L306) — the `verifyAmount(InEuint64 calldata input) external` helper with a `msg.sender != address(this)` guard, returning `FHE.asEuint64(input)`.
- Usage site: [`Z0tzPrivateLedger.sol` lines 233-235](https://github.com/0xOucan/Z0tz/blob/main/contracts/contracts/ledger/Z0tzPrivateLedger.sol#L233-L235) — line 233 is the comment `"Self-call to flip msg.sender → ledger for CoFHE input verification."`, line 235 is `euint64 amount = this.verifyAmount(op.amount);`.
- Design rationale: `FHE.asEuint64(InEuintN)` binds ACL to whatever address is `msg.sender` at the call site. When a relayer is the top-level caller, ACL incorrectly binds to the relayer, which means the consuming contract cannot later use the handle in FHE ops. The external self-call re-enters the contract so `msg.sender == address(this)` for the inner `FHE.asEuint64`, restoring the intended ACL holder. This proposal makes the pattern a one-call primitive instead of a six-line footgun every dapp rediscovers.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
