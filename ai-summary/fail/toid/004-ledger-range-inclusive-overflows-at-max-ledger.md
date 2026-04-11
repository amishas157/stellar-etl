# H002: LedgerRangeInclusive cannot represent the final supported ledger

**Date**: 2026-04-10
**Subsystem**: toid
**Severity**: Medium
**Impact**: operational correctness
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`LedgerRangeInclusive(from, to)` should return a valid half-open TOID range for every ledger value the subsystem otherwise supports individually. In particular, `LedgerRangeInclusive(2147483647, 2147483647)` should produce bounds that include the last representable ledger exactly once instead of crashing.

## Mechanism

The helper constructs the upper bound as `New(to+1, 0, 0).ToInt64()`. When `to == math.MaxInt32`, the `to+1` addition overflows `int32` before `New()` runs, producing a negative ledger sequence that `ToInt64()` rejects with `panic("invalid ledger sequence")`. Callers therefore cannot page or range-scan the final supported ledger even though standalone TOIDs for that ledger remain valid.

## Trigger

Call `LedgerRangeInclusive(2147483647, 2147483647)` or any range whose `to` endpoint is `2147483647`.

## Target Code

- `internal/toid/main.go:93-114` — inclusive range helper builds the end bound from `to+1`.
- `internal/toid/main.go:138-156` — `ToInt64()` panics when the overflowed ledger sequence becomes negative.

## Evidence

The function explicitly promises an inclusive range between two ledgers, and the rest of the package tests accept `math.MaxInt32` as a valid ledger component. But the end-bound construction needs one ledger beyond that maximum and has no overflow handling.

## Anti-Evidence

There is no current in-repo caller for `LedgerRangeInclusive`, so this may be dormant today. A reviewer should check whether external tooling or future export/pagination code relies on this helper before escalating severity.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated (distinct from fail/002 which addresses uint32-range support, not int32 MaxInt32 overflow in LedgerRangeInclusive)
**Failed At**: reviewer

### Trace Summary

Traced `LedgerRangeInclusive` at `internal/toid/main.go:96-114`. Line 112 computes `New(to+1, 0, 0)` — when `to == math.MaxInt32 (2147483647)`, Go's int32 wraparound produces `-2147483648`, and `ToInt64()` at line 141 panics on the negative ledger sequence. The bug mechanism is confirmed. However, a comprehensive grep across the entire repository shows zero production callers — `LedgerRangeInclusive` appears only in its definition and its unit test file.

### Code Paths Examined

- `internal/toid/main.go:96-114` — `LedgerRangeInclusive` confirmed: line 112 `New(to+1, 0, 0)` overflows int32 when `to == MaxInt32`
- `internal/toid/main.go:139-143` — `ToInt64()` panics on negative `LedgerSequence`, which is the result of the overflow
- `internal/toid/main_test.go:143-180` — Tests only cover small ledger values (1-3); MaxInt32 edge case is untested
- Full repo grep for `LedgerRangeInclusive` — only referenced in `main.go` (definition), `main_test.go` (test), and SKILL.md (documentation)

### Why It Failed

The overflow bug in `LedgerRangeInclusive` is real at the function level — passing `to == MaxInt32` will panic due to int32 wraparound. However, **no production code in the ETL calls this function**. The only references outside the definition are the unit test and documentation. Since no export, transform, or command code invokes `LedgerRangeInclusive`, it cannot produce wrong output or cause a panic in any current ETL pipeline. This falls under the explicit out-of-scope criterion: "Theoretical issues without a concrete code path producing wrong output." This is consistent with the precedent set by fail/003 (IncOperationOrder), which was rejected for the identical reason — a real function-level bug with zero production callers.

### Lesson Learned

When evaluating edge-case bugs in utility functions, always verify production callers before classifying as a data corruption risk. The toid package contains several helper functions (`LedgerRangeInclusive`, `IncOperationOrder`) that have genuine logic bugs but are effectively dead code in the current ETL pipeline.
