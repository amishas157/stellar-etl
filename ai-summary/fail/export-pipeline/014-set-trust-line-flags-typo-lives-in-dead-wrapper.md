# H014: `set_trust_line_flags` typo only exists in an unused duplicate details path

**Date**: 2026-04-11
**Subsystem**: export-pipeline
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

The live `history_operations` export should serialize `set_flags_s` / `clear_flags_s` with the canonical string `authorized_to_maintain_liabilities` whenever a `SetTrustLineFlags` operation touches that bit.

## Mechanism

At first glance, `transactionOperationWrapper.Details()` appears to violate that expectation because its private helper emits `authorized_to_maintain_liabilites`. If that wrapper-backed path fed the production operation export, `history_operations.details.set_flags_s` and `clear_flags_s` would silently carry the misspelled string.

## Trigger

Inspect the `SetTrustLineFlags` handling in both operation-detail builders and trace which one actually feeds `TransformOperation()`.

## Target Code

- `internal/transform/operation.go:943-955` — live `extractOperationDetails()` path uses `addTrustLineFlagToDetails()` with the correct string
- `internal/transform/operation.go:1592-1601` — duplicate `transactionOperationWrapper.Details()` path uses the misspelled helper
- `internal/transform/operation.go:2045-2077` — duplicate helper appends `authorized_to_maintain_liabilites`

## Evidence

The duplicate wrapper path really does contain the typo, and its helper diverges from the live helper earlier in the same file. That made it look like a fresh export bug at first.

## Anti-Evidence

`TransformOperation()` does not call `transactionOperationWrapper.Details()` at all; it uses `extractOperationDetails()` instead, and that path already emits the correct `authorized_to_maintain_liabilities` string. A repository-wide search for `.Details(` finds no production call site for the duplicate wrapper method, so the typo is latent dead code rather than a currently exported wrong value.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The misspelled string lives only in an unused duplicate helper. The production `history_operations` path goes through `extractOperationDetails()` and the correctly spelled `addTrustLineFlagToDetails()` helper, so no current export row uses the bad value.

### Lesson Learned

`internal/transform/operation.go` contains duplicated detail-building logic, but not every duplicate is wired into a live export surface. Before treating a divergence as a data bug, trace which builder `TransformOperation()` or `TransformEffect()` actually calls.
