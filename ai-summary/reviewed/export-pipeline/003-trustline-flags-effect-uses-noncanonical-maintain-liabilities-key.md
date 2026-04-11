# H003: `trustline_flags_updated` effects export a non-canonical maintain-liabilities key

**Date**: 2026-04-11
**Subsystem**: export-pipeline
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When a `SetTrustLineFlags` operation toggles the "authorized to maintain liabilities" bit, the resulting `trustline_flags_updated` effect should export that boolean under the same canonical key shape used by the rest of the effect payload, i.e. `authorized_to_maintain_liabilites_flag` (matching the legacy Horizon field spelling and the neighboring `authorized_flag` / `clawback_enabled_flag` keys).

## Mechanism

`setTrustLineFlagDetails()` emits the maintain-liabilities bit under `authorized_to_maintain_liabilites`, dropping the `_flag` suffix that the same effect family uses for the other trustline booleans. Downstream consumers looking for the standard effect key never see the field, so these rows silently lose that flag transition even though the exported JSON still looks structurally valid.

## Trigger

Export any ledger containing a `SetTrustLineFlags` operation that sets or clears `AuthorizedToMaintainLiabilities`. The emitted `history_effects.details` object will contain `authorized_to_maintain_liabilites: true|false` instead of the expected `authorized_to_maintain_liabilites_flag`.

## Target Code

- `internal/transform/effects.go:1127-1135` — trustline flag helper emits `authorized_flag`, `authorized_to_maintain_liabilites`, and `clawback_enabled_flag`
- `testdata/effects/10_ledgers_effects.golden:4` — checked-in golden output already contains `authorized_to_maintain_liabilites`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/protocols/horizon/effects/main.go:574-580` — upstream Horizon effect model uses `authorized_to_maintain_liabilites_flag`

## Evidence

Inside the same helper, the other trustline booleans carry `_flag` suffixes, so the maintain-liabilities field is the lone outlier. The golden fixtures confirm that production output already serializes the outlier key, which means the mismatch is user-visible today rather than a latent code-only typo.

## Anti-Evidence

The legacy Horizon spelling itself preserves the `liabilites` typo, so the spelling alone is not the core issue here. The concrete export bug is that the ETL drops the `_flag` suffix entirely, making its key shape inconsistent with both sibling fields and the legacy Horizon contract.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced the complete live effects export path: `TransformEffect()` → operation-type dispatch → `setTrustLineFlagsEffects()` / `allowTrustEffects()` → `addTrustLineFlagsEffect()` → `setTrustLineFlagDetails()`. At line 1132 of `effects.go`, the maintain-liabilities flag is emitted under key `"authorized_to_maintain_liabilites"` while its siblings use `"authorized_flag"` and `"clawback_enabled_flag"`. The upstream Horizon SDK's `TrustlineFlagsUpdated` struct uses the JSON tag `"authorized_to_maintain_liabilites_flag"` (with `_flag` suffix). Golden test fixtures confirm the wrong key is already present in production output.

### Code Paths Examined

- `internal/transform/effects.go:23` — `TransformEffect()` is the public entry point, called by the effects export command
- `internal/transform/effects.go:1094-1098` — `setTrustLineFlagsEffects()` calls `addTrustLineFlagsEffect()` with the operation's set/clear flags
- `internal/transform/effects.go:714,723,728` — `allowTrustEffects()` also calls `addTrustLineFlagsEffect()`, so the bug affects both `SetTrustLineFlags` and `AllowTrust` operations
- `internal/transform/effects.go:1101-1125` — `addTrustLineFlagsEffect()` builds details map and delegates to `setTrustLineFlagDetails()`
- `internal/transform/effects.go:1127-1137` — `setTrustLineFlagDetails()` emits three keys: `authorized_flag` (correct), `authorized_to_maintain_liabilites` (MISSING `_flag`), `clawback_enabled_flag` (correct)
- `go-stellar-sdk/protocols/horizon/effects/main.go:579` — Horizon SDK uses `authorized_to_maintain_liabilites_flag` as the canonical JSON key
- `testdata/effects/10_ledgers_effects.golden:4,8,12,...` — 20+ golden entries confirm the wrong key `authorized_to_maintain_liabilites` is emitted in production

### Findings

The bug is confirmed and actively producing incorrect output. The key `"authorized_to_maintain_liabilites"` at line 1132 is missing the `_flag` suffix that its two siblings have and that the upstream Horizon SDK expects. This affects two separate call sites: `SetTrustLineFlags` operations (line 1097) and legacy `AllowTrust` operations (lines 714, 723, 728). Any downstream system querying for the canonical key `authorized_to_maintain_liabilites_flag` will get no result, silently losing the maintain-liabilities flag transition data. The previously investigated fail/014 concerned a typo in the UNUSED `transactionOperationWrapper.Details()` path in `operation.go` — this is a distinct bug in the LIVE `effects.go` path.

### PoC Guidance

- **Test file**: `internal/transform/effects_test.go` — append a new test case
- **Setup**: Construct a `SetTrustLineFlags` operation with `AuthorizedToMaintainLiabilities` set, wrap in a minimal `LedgerTransaction`
- **Steps**: Call `TransformEffect()` and find the `trustline_flags_updated` effect in the output
- **Assertion**: Assert that `details["authorized_to_maintain_liabilites_flag"]` is present (it will be absent, confirming the bug). Also assert that `details["authorized_to_maintain_liabilites"]` is what's actually emitted (demonstrating the inconsistency).
