# H001: Synthetic signer-deletion rows zero out the removed signer's prior weight

**Date**: 2026-04-12
**Subsystem**: data-transform
**Severity**: High
**Impact**: signer-history rows lose last-known authorization weight
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When an account update removes a signer, the synthetic `account_signers` tombstone row should preserve that signer's last on-chain weight from `Pre` while marking the row as deleted. A signer removed at weight `20` should therefore export `weight = 20`, `deleted = true`, and `ledger_entry_change = REMOVED`, so downstream history queries can reconstruct the signer's final effective privileges before removal.

## Mechanism

`TransformSigners()` already diffs `Pre` vs `Post` account state to synthesize deletion rows, and it preserves other pre-image metadata such as sponsor and `LastModifiedLedger`. But for the removed signer itself it hardcodes `Weight: 0` instead of reading the actual pre-removal weight, turning a last-known-state row into a partially fabricated tombstone. That silently corrupts signer-history analytics that treat deleted rows as the final state of the removed signer.

## Trigger

1. Export `account_signers` for a ledger where an account update removes a non-master signer whose prior weight is non-zero.
2. Inspect the synthetic deletion row emitted for that signer.
3. The row will report `weight = 0` even though the removed signer had a real pre-state weight in the same `Pre` account entry.

## Target Code

- `internal/transform/account_signer.go:57-85` — removal rows are synthesized from `Pre`, but `Weight` is forced to `0`
- `internal/transform/account_signer_test.go:256-269` — checked-in expectation locks the deleted signer's weight to `0`

## Evidence

The removal branch already reads `preAccountEntry`, `preSponsors`, and `Pre.LastModifiedLedgerSeq`, proving the full pre-image is intentionally available during deletion synthesis. Despite that, the appended tombstone row discards the removed signer's prior weight and substitutes `0`, even in the unit-test fixture where the deleted signer was explicitly weight `20`.

## Anti-Evidence

The repository's current unit test explicitly expects `Weight: 0` on the synthetic deletion row, so maintainers may regard this as intentional tombstone semantics rather than last-known-state history. There is also no schema comment in `AccountSignerOutput` that states whether deleted rows must preserve prior signer weights.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated
**Failed At**: reviewer

### Trace Summary

Traced the full `TransformSigners()` function in `account_signer.go`. The deletion-synthesis branch (lines 57–86) iterates pre-state signers with `for signer := range preAccountEntry.SignerSummary()`, deliberately discarding the weight value, and hardcodes `Weight: 0` on line 78. The unit test (lines 256–269) explicitly documents this behavior with the comment "Signer B deleted: Weight=0" and asserts it as expected output. The sort order on lines 89–93 also relies on deleted rows having weight 0 for deterministic ordering.

### Code Paths Examined

- `internal/transform/account_signer.go:67` — `for signer := range preAccountEntry.SignerSummary()` deliberately discards the weight value (uses single-variable range form)
- `internal/transform/account_signer.go:75-85` — deletion tombstone row hardcodes `Weight: 0`, while preserving other pre-state fields (Sponsor, LastModifiedLedger)
- `internal/transform/account_signer_test.go:256-269` — test explicitly expects `Weight: 0` with a descriptive comment confirming this is the intended design

### Why It Failed

This is **working-as-designed tombstone semantics**, not a bug. The code treats deletion rows as representing the signer's post-deletion state (effective weight = 0, since the signer no longer exists) rather than preserving the historical last-known weight. Three independent indicators confirm this is intentional: (1) the `range` loop deliberately discards the weight value using the single-variable form, (2) the hardcoded `Weight: 0` is explicit rather than a default/zero-value accident, and (3) the unit test documents and asserts this exact behavior with explanatory comments. The hypothesis's "expected behavior" represents an alternative design choice (last-known-state history), not the actual intended semantics of the export.

### Lesson Learned

Tombstone/deletion rows in this codebase use current-state semantics (what the entity's state is *after* the change), not last-known-state semantics (what the entity was *before* the change). When a signer is removed, its effective weight is 0. The `Deleted: true` flag already communicates the removal event; `Weight: 0` reflects the entity's post-deletion reality. Do not treat this pattern as a bug in other entity types without first verifying the codebase's stated design intent via test fixtures.
