# H067: Unknown operation-type fallbacks are currently unreachable

**Date**: 2026-04-14
**Subsystem**: export-pipeline
**Severity**: Medium
**Impact**: operational correctness
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For every operation type currently emitted by Stellar ledgers, the exporter should map the type string, trace description, and details payload without error or panic.

## Mechanism

Several operation-formatting layers treat an unknown `OperationType` as fatal: `mapOperationType()` and `mapOperationTrace()` return errors, while `transactionOperationWrapper.Details()` panics. If the current protocol already had an operation arm missing from one of those switches, a legitimate transaction could either fail export entirely or emit incomplete operation metadata.

## Trigger

Process a transaction containing an operation whose `OperationType` is outside the local switch coverage.

## Target Code

- `internal/transform/operation.go:103-165` — `mapOperationType()` errors on unknown `OperationType`
- `internal/transform/operation.go:168-230` — `mapOperationTrace()` errors on unknown `OperationType`
- `internal/transform/operation.go:1364-1783` — `transactionOperationWrapper.Details()` panics on unknown `OperationType`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/xdr/xdr_generated.go:25592-25622` — current `OperationType` enum ends at `RESTORE_FOOTPRINT`

## Evidence

The operation formatter relies on large hand-maintained switches, so an omitted modern arm would be a plausible source of export failure or silently missing details if the SDK enum had already moved ahead of the local code.

## Anti-Evidence

The current generated `OperationType` enum runs from `CREATE_ACCOUNT` through `RESTORE_FOOTPRINT`, and the local switches cover that present set. I did not find a currently generated operation arm that falls into any of the unknown-type defaults.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-14
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

All currently defined `OperationType` values are covered, so the fallback error/panic paths are only latent future-compatibility hazards today.

### Lesson Learned

Large switch statements are worth auditing, but they only graduate to viable findings when the live generated enum has outrun the local case list. “Looks incomplete” is not enough without a concrete present-day missing arm.
