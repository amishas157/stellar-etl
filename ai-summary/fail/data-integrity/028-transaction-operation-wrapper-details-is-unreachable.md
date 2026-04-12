# H028: `transactionOperationWrapper.Details()` looks like a better formatter, but it is unreachable for live exports

**Date**: 2026-04-12
**Subsystem**: data-integrity
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If the repository contains an alternate operation-detail formatter that preserves more exact values or more consistent types, the live `history_operations` export would need to call that formatter (or reuse its logic) for any divergence to affect shipped output.

## Mechanism

`transactionOperationWrapper.Details()` builds a second operation-details map with noticeably different typing from `extractOperationDetails()`, including exact `amount.String(...)` values in several path-payment branches. That initially looked like evidence that the live exporter had bypassed a more correct implementation.

## Trigger

1. Trace the code path used by `export_operations`.
2. Compare it with the legacy `transactionOperationWrapper.Details()` implementation.
3. Determine whether any production caller actually routes operation rows through `Details()`.

## Target Code

- `cmd/export_operations.go:25-43` — live CLI path calls `transform.TransformOperation(...)`
- `internal/transform/operation.go:TransformOperation:29-57` — live transformer delegates to `extractOperationDetails(...)`
- `internal/transform/operation.go:transactionOperationWrapper.Details:1363-1435` — alternate formatter with different amount typing
- `internal/transform/effects.go:31-42` — the wrapper is instantiated for effects generation, but that path calls `operation.effects()`, not `operation.Details()`

## Evidence

The duplicate formatter is real and materially different from the live `extractOperationDetails()` implementation, so it initially looked like proof that a precision-preserving alternative already existed in the codebase.

## Anti-Evidence

The actual CLI operation export path never calls `transactionOperationWrapper.Details()`. The only live wrapper usage I found is the effects pipeline, which invokes `operation.effects()` instead, leaving `Details()` as dormant code for current exports.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The alternate formatter does not participate in any production export path that writes `history_operations`. A divergence in unreachable helper code cannot produce wrong output on its own, so it is not a live data-integrity bug.

### Lesson Learned

When duplicate helpers suggest a "better path exists" argument, always trace the live call graph before promoting it to a hypothesis. In this repository, `transactionOperationWrapper` is still live for effects, but `Details()` itself is not part of the shipped operation-export pipeline.
