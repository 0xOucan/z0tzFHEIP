# Tezcatli × Z0tz integration — confidential DeFi with on-chain compliance

> Living document. Updated whenever the integration's public surface changes.
> Last revised: 2026-05-04 (cofhesdk-update branch merged to main).

## Why this integration

Z0tz needed a yield surface that:

1. Doesn't break the wallet's privacy model — the user's smart account must never appear in the vault's events, share allocations, or strategy adapter calls.
2. Carries an on-chain AML / OFAC / KYC choke point so production deployments can flip enforcement on without a contract upgrade.
3. Returns chain-truth state that's reproducible from the user's passkey alone, so wallet UX doesn't need a persistent sidecar to show numbers.

Tezcatli's confidential vault stack hits all three. This document describes how Z0tz consumes it, what the interface contract is, and what happens at every step of a deposit / withdraw.

## Topology

```
Z0tz wallet (passkey)
   │
   ├─ V6.5 ledger (encrypted balance)              ◄── user's home
   │
   ▼
HKDF DeFi stealth (per origin × vault × index)     ◄── one-time proxy
   │
   ▼
tzcUSDC wrapper (FHERC-20)                         ◄── shield / unshield
   │
   ▼
TezcatliConfidentialVault                          ◄── records deposit
   │
   ├─ Z0tzComplianceGate (FHEIP-0010)              ◄── canShield / canUnshield
   │     ├─ MockZ0tzKycRegistry (KYC attestations)
   │     ├─ MockOFACSanctionsList (OFAC oracle)
   │     └─ Z0tzDepositorRegistry (audit trail)
   │
   └─ TezcatliStrategyAdapterAaveV3                ◄── coordinator-driven
         │
         ▼
      Aave V3 supply (yield)
```

## On-chain primitives

All addresses below are arb-sepolia (the only Aave V3 chain in the testnet matrix). Per-chain registry: `contracts/deployments/defi-vaults.json`.

| Component | Purpose | Z0tz field name |
|---|---|---|
| `TezcatliConfidentialVault` | The vault itself. Holds encrypted FHERC-20 shares, exposes `confidentialSharesOf` / `principalDepositedOf` / `netPositionSnapshotOf` / `pendingYieldSnapshotOf` per FHEIP-0011. | `vault.vault` |
| `TezcatliWrappedToken` (`tzcUSDC`) | FHE wrapper around USDC. Shield → encrypted balance; unshield → plaintext. The vault's `asset()`. | `vault.wrapped` |
| `TezcatliStrategyAdapterAaveV3` | Adapter that supplies USDC to Aave V3, holds the resulting aTokens, redeems on demand. Only the vault's `coordinator` can deploy/redeem. | live read: `vault.strategyAdapter()` |
| `Z0tzComplianceGate` | FHEIP-0010 reference implementation. `canShield(account, periodTag, amount) → (bool, uint8)` consulted on every deposit; `canUnshield(...)` on every withdraw. Default-permissive (`enabled=false`); admin flips to enforced via `setEnabled(true)`. | `vault.complianceGate()` |
| `MockZ0tzKycRegistry` | Yes/no KYC attestation per address. Owned by the gate admin. Tezcatli deployments use this in default-permissive mode for testnet; production deployments swap in a verified registry. | `gate.kycRegistry()` |
| `MockOFACSanctionsList` | Sanctions blocklist consulted in `canShield` / `canUnshield` before the gate's own deny-list. | `gate.ofacOracle()` |
| `Z0tzDepositorRegistry` | Append-only on-chain record of every screened address that has ever deposited. Audit-trail surface. | (per-deployment, not exposed on the gate) |

The `z0tz defi vaults [--chain]` CLI command probes each vault on chain, dumps the four addresses (`vault.complianceGate()`, `gate.kycRegistry()`, `gate.ofacOracle()`, `vault.strategyAdapter()`), and flags any drift between the registry JSON and live state.

## Stealth derivation

Z0tz's HKDF derivation extends to a fourth family — **DeFi stealths** — keyed on `(passkey, originChainId, vaultChainId, vaultAddress, index)`.

The `originChainId` baked into the derivation is what makes cross-chain DeFi **frictionless on withdraw**: a position deposited from base-sepolia → arb-sepolia vault lives at one stealth, a position deposited from arb-sepolia → arb-sepolia vault lives at a different stealth, and the scanner can iterate origins to discover any of them. When the user clicks Withdraw, the renderer reads `position.originChainId` and routes funds back to that chain via CCTP automatically.

Reference: `cli/src/ledger/idDerivation.ts:deriveDefiStealth(...)` — HKDF salt packs both chain IDs, info string is `"z0tz-defi-stealth" || vault || index`.

## Flows

### Same-chain deposit (arb ledger → arb vault)

1. **Smart-fit alice selection.** Decrypt every live ledger entry, pick the smallest one that covers the requested amount. Bails with `Insufficient single-entry balance` if none cover.
2. **Cashout.** `Z0tzPrivateLedger.spend(Cashout)` debits alice, mints encrypted FHERC-20 to the DeFi stealth's wrapper balance (in `wrappedUsdcV5`, the V6.5 wrapper).
3. **Unshield.** Stealth calls `wrappedUsdcV5.unshield(stealth, amount)` — burns encrypted, queues claim.
4. **TN decrypt + claim.** Stealth calls `claimUnshielded(ctHash, decValue, sig)` — wrapper sends plaintext USDC to stealth.
5. **Compliance preflight.** GUI calls `gate.canShield(stealth, 0, amount)` via `eth_call`. If denied, abort with friendly reason code (KYC required, OFAC, daily cap, etc.) — no further gas burned.
6. **Approve + shield + deposit.**
   - `usdc.approve(tzcUSDC, amount)`
   - `tzcUSDC.shield(amount)` — wrapper takes USDC, mints encrypted tzcUSDC to stealth.
   - `tzcUSDC.confidentialTransferAndCall(vault, encAmount, abi.encode(stealth))` — vault's `onConfidentialTransferReceived` records the deposit, calls `_recordDeposit` which updates `principalDepositedOf` + `netPositionSnapshotOf` + share math.
7. **On-deposit strategy activation.** Z0tz client POSTs `/api/strategy-deploy` to the relayer. The relayer (= on-chain coordinator) calls `vault.coordinatorDeployToStrategy(adapter, idle, minSharesOut)` — wrapper releases USDC, adapter supplies to Aave V3, vault credits aTokens to `strategySharesByAdapter[adapter]`.

### Cross-chain deposit (base ledger → arb vault)

Same as above but inserts a CCTP V2 burn-and-mint between steps 4 and 5. The destination of the mint is the HKDF DeFi stealth on arb derived with `originChainId = 84532` so the position carries its origin in its identity.

### Withdraw — origin-aware auto-route

1. **Pre-redeem from Aave.** Z0tz client POSTs `/api/strategy-redeem` to the relayer with the requested amount. The relayer reads `usdc.balanceOf(tzcUSDC)` (the real wrapper reserve, NOT `idleAssetsHint` which over-counts after withdraws), computes shortfall, calls `vault.coordinatorRedeemFromStrategy(adapter, shares, minAssetsOut)`. Adapter pulls aTokens from Aave, vault re-shields the resulting USDC into the wrapper.
2. **Compliance preflight.** `gate.canUnshield(...)` via `eth_call`.
3. **Withdraw.** `vault.withdrawConfidential(stealth)` — vault transfers encrypted shares from itself to the stealth.
4. **Unshield.** `tzcUSDC.unshield(amount)` — wrapper sends plaintext USDC to stealth.
5. **Route home.** If `position.originChainId == vault.chainId`, sweep directly into the user's V6.5 ledger on the same chain. Otherwise, CCTP burn at the stealth → mint at an HKDF-derived stealth on the origin chain → `privateSweepToLedger`.

The withdraw modal's Destination dropdown defaults to the origin chain but the user can override (e.g., a `from base` position can be withdrawn directly to arb if they want). Both chains pre-flight the compliance gate.

## Compliance gate behavior

The gate's `canShield` / `canUnshield` returns `(bool allowed, uint8 reasonCode)`. Reason codes are defined in FHEIP-0010 §3:

| Code | Meaning |
|---|---|
| 0 | Default deny (admin hasn't enabled this op) |
| 1 | KYC verification required |
| 2 | OFAC / AML block-list hit |
| 3 | Daily / per-tx amount cap exceeded |
| 4 | Amount below configured minimum |
| 5 | Asset not whitelisted |
| 6 | Geofence / jurisdiction restriction |
| 7 | Report required, but allow (vault emits `ComplianceReportRequired`) |

Z0tz's GUI translates each code into a human-readable error string — see `gui/src/main/ipc-handlers.ts:explainDefiError`. The relayer never holds compliance state.

The gate is **default-permissive** on testnet: `enabled=false` short-circuits to allow. Production turns enforcement on with a single owner tx (`gate.setEnabled(true)`).

## Display contract — what the wallet shows per position

After the no-cache refactor (commit `29afd2c`), the GUI surfaces four numbers per active position:

| Row | Source | Behavior |
|---|---|---|
| **Withdrawable** | `min(netPositionSnapshotOf + APY × elapsed-since-snapshot, aTokenBalance(adapter) + usdc.balanceOf(wrapper))` | Live estimate, capped at the vault's actual recoverable USDC so the displayed number always matches what `unshield` can pay out. |
| **Deposited** | `principalDepositedOf` (decrypted via stealth's CoFHE permit) | Cumulative cost basis recorded on chain at every deposit. Reproducible from passkey alone. |
| **Yield** | Withdrawable − Deposited | Includes the live extrapolation, so Aave accrual between snapshots is visible. |
| **APY** | Aave V3 supply rate | Read directly from Aave's reserve data. |

Closed positions (`hasActivePosition() = false`) are filtered at the scanner so the active list stays clean.

There is no local cost-basis cache, no JSON sidecar, no React state collision. Every render fetches from chain.

## Operator playbook

| Task | How |
|---|---|
| Verify on-chain compliance wiring | `z0tz defi vaults` — dumps `complianceGate` / `kycRegistry` / `ofacOracle` / `strategyAdapter`, flags drift |
| Check current Aave deposit | `aToken.balanceOf(adapter)` via the strategy-deploy or strategy-redeem endpoint response |
| Enable enforcement | Owner calls `gate.setEnabled(true)` (one tx per chain) |
| Add KYC attestation | Owner calls `kycRegistry.attest(addr)` (subject to gate config) |
| Add OFAC entry | Owner calls `ofacOracle.addToBlocklist(addr)` |
| Relax allocation cap (initial deploy) | `npx hardhat run scripts/relax-tezcatli-allocation-cap.ts --network arb-sepolia` |
| Manual force-deploy idle | Hit `/api/strategy-deploy` with `{chainId: 421614}` (passkey-authed) |
| Manual force-redeem | Hit `/api/strategy-redeem` with `{chainId: 421614, amountUsdc: "X"}` |

## Open issues

- **Multi-source spend.** `Ledger.spend` debits one alice per call. When a user wants to spend more than their largest single alice can cover, they currently get a clear error and use `LEDGER_INTERNAL` to merge first. A native `spendMulti(SpendOp[])` would atomically debit N entries — scoped for the next protocol revision.
- **Idle bucket drift.** Tezcatli's `idleAssetsHint` is documented as a lower bound that never decrements on user withdraws. Z0tz's strategy-redeem endpoint sources from `usdc.balanceOf(wrapper)` instead, which is exact. A future Tezcatli revision could expose a real-idle view.
- **Share-pricing windfalls.** When `coordinatorRedeemFromStrategy` runs, `_internalAssets += assetsOut` re-prices existing shares upward. In multi-user vaults this can transfer value from stagnant positions to active ones. Z0tz caps displayed withdrawable at `aTokenBalance + wrapperReserve` to keep users honest about what they can actually pull out, but the underlying contract behavior is by design and would require a separate FHEIP to address.

## References

- [FHEIP-0010](./FHEIP-0010-confidential-vault-compliance-gate.md) — compliance gate interface
- [FHEIP-0011](./FHEIP-0011-confidential-vault-position-snapshots.md) — per-account encrypted position snapshots
- [FHEIP-0012](./FHEIP-0012-compliance-aware-cash-in-cash-out.md) — wallet-side cash-in/out compliance
- `contracts/contracts/tezcatli/` — Tezcatli vault + adapter sources
- `contracts/contracts/compliance/` — Z0tz compliance primitives (gate, KYC, OFAC, depositor registry)
- `contracts/scripts/relax-tezcatli-allocation-cap.ts` — admin script for the first-deploy bootstrap
- `landing/app/api/strategy-deploy/route.ts` — relayer endpoint for on-deposit Aave activation
- `landing/app/api/strategy-redeem/route.ts` — relayer endpoint for pre-withdraw Aave redemption
